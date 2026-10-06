# Genesys AudioHook & Transcription Connector Server & LLM Translation

This repository contains an implementation of a Genesys AudioHook & Transcription Connector server that processes real-time audio streams for transcription using either **Google Cloud Speech-to-Text API** or **OpenAI Speech-to-Text API** and translation using **Google Gemini**. The transcribed (and optionally translated) text is then injected back into Genesys Cloud via event messages. This server is designed to meet the Genesys AudioHook protocol requirements and supports essential transactions such as session establishment, audio streaming, ping/pong heartbeats, and clean disconnection.

The project is designed to be deployed on **Digital Ocean** (or a similar platform) and integrates with **Google Cloud** or **OpenAI** for transcription and **Google Gemini** for translation.

---

## Table of Contents

- [Overview](#overview)
- [Use Cases](#use-cases)
- [Architecture](#architecture)
- [Code Structure](#code-structure)
- [Transcription and Translation Processing](#transcription-and-translation-processing)
- [Supported Speech Models](#supported-speech-models)
- [Language Handling](#language-handling)
- [Dynamic Transcription Vendor Selection](#dynamic-transcription-vendor-selection)
- [Synthetic Timestamps and Confidence Scores](#synthetic-timestamps-and-confidence-scores)
- [Deployment](#deployment)
  - [Digital Ocean App Platform Configuration](#digital-ocean-app-platform-configuration)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Error Handling and Logging](#error-handling-and-logging)
- [Configuration](#configuration)
- [Known Issues](#known-issues)

---

## Overview

The server accepts WebSocket connections from Genesys Cloud (the AudioHook client) and performs the following key operations:

1.  **Connection Establishment & Validation:**
    - Validates incoming `HTTPS` upgrade requests against required headers (e.g., `API key`, `organization ID`).
    - Negotiates a media format (typically `PCMU` at 8000 Hz).

2.  **Session Lifecycle Management:**
    - Manages the session lifecycle by handling `"open"`, `"ping"`, `"close"`, and other transaction messages.
    - Sends an `"opened"` message to Genesys Cloud upon successful open transaction, enabling audio streaming.

3.  **Real-Time Audio Processing and Control Message Handling:**
    - Processes incoming audio frames (in `PCMU` format) in real time.
    - Converts audio frames from `PCMU` (u-law) to `PCM16` using the Python `audioop` module.
    - **Control Message Handling:** The server processes control messages—such as `"paused"`, `"discarded"`, and `"resumed"`—to adjust the effective audio timeline. This ensures that the computed offsets for transcription events exclude any periods where audio was lost or intentionally paused, aligning with the Genesys AudioHook protocol requirements.

4.  **Transcription via Google Cloud Speech-to-Text or OpenAI Speech-to-Text:**
    - Sends `PCM16` audio to either **Google Cloud Speech-to-Text API** or **OpenAI's Speech-to-Text API** for transcription in the source language.
    - The transcription vendor can be specified in the open message via `customConfig.transcriptionVendor`, with fallback to the environment variable.

5.  **Translation via Google Gemini (Optional):**
    - If enabled via `customConfig.enableTranslation` in the open message, translates the transcribed text to the destination language using **Google Gemini** (see 'enableTranslation' in 'Language Handling' section).
    - Uses structured output to ensure only the translated text is returned.
    - If disabled or not specified, the original transcript is returned without translation, using the input language.

6.  **Injection Back into Genesys Cloud:**
    - Constructs a **transcript event message** with the (translated or original) text, including accurate **offset** and **duration** values adjusted for any control messages.
    - Sends the message back to Genesys Cloud via the WebSocket connection for injection into the conversation.

---

## Use Cases

This connector is designed to support two primary use cases that address different needs in contact center environments:

### 1. Transcription Only (No Translation)

This use case is ideal when you need a specialized transcription engine different than the native options provided by Genesys Cloud or EVTS.

**Key benefits:**
- Leverage either Google's or OpenAI's advanced speech recognition capabilities
- Supports languages that might not be available in Genesys' native transcription or EVTS

**Configuration:**
- Set `enableTranslation: false` or omit it in the `customConfig`
- Ensure the `inputLanguage` in `customConfig` matches the language being spoken

This approach maintains the original language throughout the conversation, making it suitable for environments where all systems (including analytics, agent assistance, etc.) support the source language.

### 2. Transcription + Translation

This use case is particularly valuable for enabling advanced Genesys features (like Copilot or Speech & Text Analytics) for languages that aren't directly supported by these tools.

**Example scenario:**
A contact center serves customers who speak a regionally-important language (such as Basque, Zulu, Welsh, etc.) that isn't directly supported by Genesys Copilot or STA. However, these tools do support a widely-used language in that same region (such as Spanish or English).

**How it works:**
1.  The customer speaks in their preferred language (e.g., Basque)
2.  The connector transcribes the audio in the source language
3.  The text is translated to a widely-supported language (e.g., Spanish)
4.  Genesys Cloud receives the translated transcript, enabling tools like Copilot and STA to function

**Key benefits:**
- Extends advanced Genesys features to additional languages
- Provides a more inclusive customer experience
- Leverages existing agent language capabilities
- Enables analytics and assistance tools across more languages

**Configuration:**
- Set `enableTranslation: true` in the `customConfig`
- Set `inputLanguage` to the regionally-important language (source)
- The `language` field in the message determines the target language for translation

This use case is especially valuable in regions with linguistic diversity, where contact centers need to support regional languages while leveraging tools optimized for more widely-spoken languages.

---

## Architecture

The application is built around the following core components:

-   **WebSocket Server:**
    -   Uses the `websockets` library to manage connections and message exchanges with Genesys Cloud.

-   **Session Handler (`AudioHookServer`):**
    -   Processes incoming messages, handles transactions (open, ping, close, etc.), manages rate limiting, and adjusts transcription offsets based on control messages.
    -   Implemented in `audio_hook_server.py`.

-   **Audio Processing:**
    -   Converts audio frames from `PCMU` to `PCM16` using `audioop`.
    -   Feeds `PCM16` audio to either Google Cloud Speech-to-Text or OpenAI Speech-to-Text for transcription.
    -   Optionally translates transcribed text using Google Gemini.

-   **Transcription and Translation:**
    -   **Transcription:** Uses either:
        -   Google Cloud Speech-to-Text API with streaming recognition for real-time transcription.
        -   OpenAI Speech-to-Text API with buffered streaming for real-time transcription.
    -   **Translation (Optional):** Uses Google Gemini with structured output to ensure only the translated text is returned. This step is performed only if `customConfig.enableTranslation` is set to true in the open message.

-   **Rate Limiting:**
    -   Implements a custom rate limiter to prevent exceeding GC AudioHook's messaging rate limits.
    -   Defined in `rate_limiter.py`.

-   **Environment Configuration:**
    -   Loads configurations (API keys, Google Cloud settings, OpenAI settings, rate limits, supported languages, etc.) from environment variables.
    -   Managed in `config.py`.

---

## Code Structure

-   **`Procfile`**
    Specifies the command to start the application:
    ```bash
    web: python main.py
    ```

-   **`main.py`**
    -   Main entry point that starts the WebSocket server.
    -   Validates incoming connections and delegates handling to `AudioHookServer`.
    -   Includes WebSocket handshake validation and health endpoint (`/health`) for Digital Ocean.

-   **`audio_hook_server.py`**
    -   Contains the `AudioHookServer` class, which manages:
        -   Session lifecycle (open, ping, close, etc.).
        -   Audio frame processing, control message handling, and rate limiting.
        -   Transcription and (optionally) translation event sending back to Genesys Cloud.
        -   Dynamic loading of transcription providers based on `customConfig`.
        -   For probe connections, the server sends the list of supported languages (as defined in the `SUPPORTED_LANGUAGES` environment variable) to Genesys Cloud.
        -   The server adjusts transcript offsets based on control messages (`"paused"`, `"discarded"`, and `"resumed"`) to ensure that only the processed audio timeline is considered.

-   **`google_speech_transcription.py`**
    -   Implements the `StreamingTranscription` class for real-time transcription using Google Cloud Speech-to-Text.
    -   Handles audio conversion from `PCMU` to `PCM16` and feeds it to the API.
    -   Includes `normalize_language_code` for BCP-47 language code normalization.

-   **`openai_speech_transcription.py`**
    -   Implements the `StreamingTranscription` class for real-time transcription using OpenAI's Speech-to-Text API.
    -   **Features:**
        -   Intelligent buffering system that accumulates audio until complete utterances are detected
        -   Voice Activity Detection (VAD) to identify speech segments and silence
        -   Creates temporary WAV files for processing detected utterances
        -   Streams data to OpenAI's API with appropriate parameters for real-time performance
        -   Processes response chunks to build complete transcripts with confidence scores
        -   Includes language code mapping from BCP-47 to ISO formats
        -   Generates synthetic word-level timing information for compatibility with Genesys AudioHook
        -   Advanced token-to-word confidence mapping for accurate per-word confidence scores
        -   Artifact filtering to prevent spurious words/phrases in transcripts
        -   Initial frame skipping to avoid connection sounds/beeps
        -   Low confidence token filtering

-   **`google_gemini_translation.py`**
    -   Implements the `translate_with_gemini` function for translating text using Google Gemini.
    -   Uses structured output (via Pydantic) to ensure only the translation is returned.
    -   Handles translation errors and logs them appropriately.

-   **`language_mapping.py`**
    -   Contains functions for normalizing language codes:
        -   `normalize_language_code`: Normalizes language codes to BCP-47 format (e.g., "es-es" → "es-ES").
        -   `get_openai_language_code`: Maps BCP-47 codes to ISO 639-1/639-3 codes compatible with OpenAI's API.
        -   `is_openai_unsupported_language`: Identifies languages not directly supported by OpenAI's API.
        -   `get_language_specific_prompt`: Provides native language prompts for unsupported languages.

-   **`rate_limiter.py`**
    -   Provides an asynchronous rate limiter (`RateLimiter`) to throttle message sending.
    -   Supports Genesys Cloud's rate limits (e.g., 5 messages/sec, 25 burst limit).

-   **`config.py`**
    -   Loads all configuration variables from environment variables.
    -   Includes settings for Google Cloud, OpenAI, Google Gemini, Genesys, rate limiting, and supported languages.

-   **`utils.py`**
    -   Contains helper functions:
        -   `format_json`: Pretty-prints JSON for logging.
        -   `parse_iso8601_duration`: Parses ISO 8601 duration strings for rate limiting.

-   **`requirements.txt`**
    -   Lists all Python dependencies required for the project.

---

## Transcription and Translation Processing

-   **Receiving Audio:**
    -   Genesys Cloud streams audio frames (binary WebSocket messages) after the open transaction.
    -   Each frame is received in `AudioHookServer.handle_audio_frame`.

-   **Real-Time Processing:**
    -   Converts audio frames from `PCMU` (u-law) to `PCM16` using `audioop`.
    -   Supports multi-channel audio (e.g., stereo, with external and internal channels).

-   **Control Message Handling:**
    -   The server processes control messages such as `"paused"`, `"discarded"`, and `"resumed"`.
    -   These messages adjust an internal offset (tracked as `processed_audio_samples`) so that transcription offsets and durations accurately reflect only the audio that was received (excluding any gaps due to pauses or audio loss).

-   **Transcription Provider Selection:**
    -   The system dynamically selects the appropriate transcription provider based on `customConfig.transcriptionVendor` with fallback to the `DEFAULT_SPEECH_PROVIDER` environment variable.
    -   This selection determines which implementation of `StreamingTranscription` is instantiated.

-   **Google Cloud Transcription:**
    -   Uses Google Cloud Speech-to-Text API with streaming recognition.
    -   Feeds `PCM16` audio directly to the API.
    -   Retrieves transcription results with word-level timing and confidence scores when available.

-   **OpenAI Transcription:**
    -   Uses an intelligent utterance detection system:
        -   Accumulates audio frames in a buffer
        -   Applies Voice Activity Detection (VAD) to identify speech segments
        -   Detects end of utterances based on silence duration (currently 800ms)
        -   Creates temporary WAV files for complete utterances
        -   Sends audio to OpenAI's API via streaming for real-time results
        -   Process chunked responses to build complete transcriptions
        -   Extracts confidence scores from logprobs when available
        -   Maps token-level confidence to word-level confidence
        -   Generates synthetic word-level timing since OpenAI doesn't provide it
    -   Implements safeguards:
        -   Timeout-based processing to ensure audio doesn't accumulate indefinitely
        -   Energy thresholds to avoid processing silence
        -   Buffer overflow prevention
        -   Duplicate transcript prevention
    -   Transcript Quality Improvements:
        -   Skips initial audio frames to avoid connection sounds/beeps
        -   Filters out known spurious artifacts ("context:", "ring", etc.)
        -   Uses regex pattern matching to identify and remove common artifacts
        -   Filters low-confidence tokens that might represent misinterpreted sounds
        -   Uses enhanced prompting to instruct the model to ignore system sounds

-   **Translation (Optional):**
    -   If `customConfig.enableTranslation` is set to `true` in the open message, the transcribed text is sent to Google Gemini for translation into the destination language.
    -   If disabled or not specified, the original transcript is returned without translation, using the input language.
    -   Structured output ensures that only the translated (or original) text is returned.
    -   Translation failures are logged and skipped.

-   **Injection Back into Genesys Cloud:**
    -   Constructs a transcript event message with:
        -   Unique transcript ID.
        -   Channel identifier (e.g., 0 for external, 1 for internal).
        -   Transcribed text with adjusted offsets, duration, and confidence.
    -   Sends the event to Genesys Cloud via WebSocket for conversation injection.

---

## Supported Speech Models

This connector supports two speech recognition providers:

**Google Cloud Speech-to-Text**

### Chirp 2
-   The most advanced model with full feature support, including:
    -   Greater performance
    -   Faster
    -   Word-level confidence scores
-   Limited language support

### Chirp
-   Good model with broad language support:
    -   Does not support word-level confidence scores (fixed value of `1.0` is used)
    -   Slower, a bit more lag to get the transcript back into GC

**OpenAI Speech-to-Text**

### gpt-4o-mini-transcribe
-   Default model, balancing speed and accuracy
-   Limited parameter support (no timestamps)
-   Uses a sophisticated buffering system to detect complete utterances
-   **Features:**
    -   Voice Activity Detection to process only speech segments
    -   Response streaming for real-time results
    -   Confidence scores derived from token logprobs
    -   Synthetic word-level timing for Genesys compatibility
    -   Artifact filtering to prevent spurious transcripts

### gpt-4o-transcribe
-   Higher quality model for more accurate transcriptions
-   Limited parameter support (no timestamps)
-   Same processing features as `gpt-4o-mini-transcribe`

The connector automatically adapts to whichever provider and model is specified in the environment variables, adjusting request parameters and response handling accordingly. When using models without word-level confidence or timing, the connector still maintains full compatibility with the Genesys AudioHook protocol by supplying generated values where needed.

---

## Language Handling

-   **Input Language (Source):**
    -   Determined from the `customConfig.inputLanguage` field in the `"open"` message received from Genesys Cloud. For example:
      ```json
      {
        "inputLanguage": "es-es",
        "enableTranslation": true
      }
      ```
    -   Used for transcription via Google Cloud Speech-to-Text or OpenAI Speech-to-Text.
    -   Defaults to `"en-US"` if not provided.
    -   Normalized to BCP-47 format using `normalize_language_code`.

-   **Language Code Mapping for OpenAI:**
    -   OpenAI's speech models support ISO 639-1/639-3 language codes rather than BCP-47 format.
    -   The connector automatically maps BCP-47 codes (e.g., `"es-ES"`) to ISO codes (e.g., `"es"`) before sending to OpenAI using the `get_openai_language_code` function.
    -   This mapping covers all major language variants (Spanish, English, French, etc.) and gracefully handles unsupported codes.
    -   This mapping is handled transparently, so you can continue using BCP-47 codes in your Genesys configuration.

-   **Unsupported Languages Handling:**
    -   For languages not officially supported by OpenAI's API (like Zulu/`zu-ZA`), the connector uses a special approach:
        -   Detects unsupported languages using the `is_openai_unsupported_language` function
        -   Omits the `language` parameter that would cause API errors
        -   Instead, includes a native language prompt (e.g., `"Humusha ngesizulu (Mzansi Afrika)"` for Zulu)
        -   The prompt is provided in the target language to help guide the model appropriately
    -   This approach allows transcription in languages that OpenAI doesn't explicitly support via their `language` parameter.

-   **Destination Language:**
    -   Determined from the `language` field in the `"open"` message.
    -   Used as the target language for translation via Google Gemini when translation is enabled.
    -   Normalized to BCP-47 format.

-   **Supported Languages:**
    -   Defined in the `SUPPORTED_LANGUAGES` environment variable (comma-separated, e.g., `"es-ES,it-IT,en-US"`).
    -   Sent to Genesys Cloud in the `"opened"` message for probe connections.

-   **Translation Toggle:**
    -   The `customConfig.enableTranslation` boolean in the `"open"` message controls whether translation is enabled for the session.
    -   If disabled or not specified, the server returns the original transcription without translation, using the input language.

---

## Dynamic Transcription Vendor Selection

The server now supports dynamic selection of the transcription vendor on a per-conversation basis:

-   **Configuration in Genesys Open Message:**
    -   The transcription vendor can be specified in the `"open"` message via the `customConfig.transcriptionVendor` field:
      ```json
      {
        "transcriptionVendor": "google",  // or "openai"
        "inputLanguage": "es-es",
        "enableTranslation": true
      }
      ```
    -   This allows different conversations to use different transcription providers based on specific needs.

-   **Default Fallback:**
    -   If not specified, the server falls back to using the `DEFAULT_SPEECH_PROVIDER` environment variable.
    -   This maintains backward compatibility with existing deployments.

-   **Dynamic Module Loading:**
    -   The server uses Python's `importlib` module to dynamically load the appropriate transcription provider at runtime.
    -   The `_load_transcription_provider()` method instantiates the correct module after receiving the `"open"` message.

-   **Fault Tolerance:**
    -   If the specified provider fails to load, the system gracefully falls back to the Google provider.
    -   This ensures robustness even if configuration errors occur.

-   **Benefits:**
    -   Enables A/B testing between different providers
    -   Allows different language needs to be serviced by different providers
    -   Creates flexibility to use the most appropriate provider for specific scenarios
    -   Eliminates the need for multiple deployments for different provider needs

---

## Synthetic Timestamps and Confidence Scores

### Synthetic Timestamps for OpenAI

Since OpenAI's Speech-to-Text API doesn't provide word-level timing information, the connector generates synthetic timestamps to ensure compatibility with Genesys AudioHook:

-   **Audio Position Tracking:**
    -   The system tracks the accurate position of audio in the stream, accounting for paused and discarded segments.
    -   This ensures that even synthetic timestamps accurately reflect the true audio timeline.

-   **Utterance-Based Timestamp Generation:**
    -   When speech is detected, the system records the position where the utterance begins.
    -   This position is used as the base timestamp for all words in the utterance.
    -   The utterance duration is calculated based on the number of audio samples processed.

-   **Word-Level Distribution:**
    -   The total utterance duration is evenly distributed across all words in the transcript.
    -   Each word is assigned a precise start and end time relative to the utterance start.
    -   For example, with a 3-second utterance containing 6 words, each word would be allocated 0.5 seconds.

-   **Timeline Adjustment:**
    -   Timestamps are adjusted based on `"paused"` and `"discarded"` messages.
    -   This ensures that reported timestamps exclude periods of silence or discarded audio.
    -   The `offset_adjustment` property tracks the number of samples to adjust in the timeline.

-   **Accurate Temporal Alignment:**
    -   The result is synthetic timestamps that closely align with the actual speech.
    -   This provides a realistic visualization in the Genesys Cloud UI.
    -   The timestamps properly account for the complete audio timeline, including gaps and pauses.

### Confidence Scores

The connector ensures proper confidence scores for all transcription models:

-   **Google Chirp 2:**
    -   Uses native word-level confidence scores directly from the API
    -   These scores represent the model's certainty for each word
    -   Values range from `0.0` to `1.0` with higher values indicating greater confidence

-   **Google Chirp:**
    -   Since Chirp doesn't support word-level confidence scores (always returns `0.0`)
    -   The connector automatically replaces these `0.0` values with `1.0`
    -   This ensures proper display in Genesys Cloud UI and prevents errors
    -   This fallback applies only to words with explicitly zero confidence scores

-   **OpenAI models:**
    -   Derives confidence scores from token logprobs
    -   Maps token-level confidence to word-level confidence using advanced matching
    -   Provides realistic variation in confidence based on the model's certainty
    -   Reflects lower confidence for unusual terms or unclear audio

This approach ensures consistent and meaningful confidence scores across all providers and models, maintaining full compatibility with the Genesys AudioHook protocol even when using models with limited confidence score support.

---

## Deployment

This project is designed to be deployed on **Digital Ocean** (or a similar platform). It integrates with **Google Cloud** or **OpenAI** for transcription (Speech-to-Text API) and **Google Gemini** for translation.

### Digital Ocean App Platform Configuration

When deploying this application on Digital Ocean App Platform, you'll need to configure the following settings:

-   **HTTP Request Routes**
    -   **Route Path:** `/audiohook`
    -   **Preserve Path Prefix:** Enabled (check this option to ensure the path will remain `/audiohook` when forwarded to the component)

-   **Ports**
    -   **Public HTTP Port:** `443` (for HTTPS connections)

-   **Health Checks**
    -   **Path:** `/health`
    -   **Protocol:** HTTP

-   **Commands**
    -   **Build Command:** None
    -   **Run Command:** `python main.py`

These settings ensure that:
-   The application listens on the correct path (`/audiohook`) for incoming Genesys Cloud AudioHook connections.
-   The health check path (`/health`) is properly configured to allow Digital Ocean to monitor the application's status.
-   The application starts correctly with the proper run command.

**Important:** When configuring your Genesys Cloud AudioHook integration, use the full URL provided by Digital Ocean (e.g., `https://startish-app-1gxm4.ondigitalocean.app/audiohook`) as your connector endpoint.

---

## Prerequisites

-   **Dependencies:**
    -   All Python dependencies are listed in `requirements.txt`:
        -   `websockets`
        -   `aiohttp`
        -   `pydub`
        -   `python-dotenv`
        -   `google-cloud-speech`
        -   `google-generativeai`
        -   `openai`

-   **Google Cloud Account:**
    -   Required for Google Cloud Speech-to-Text API access if using Google as the speech provider.
    -   Set up a service account and download the JSON key.

-   **OpenAI API Key:**
    -   Required for OpenAI Speech-to-Text API access if using OpenAI as the speech provider.
    -   Obtain from OpenAI's platform.

-   **Google Gemini API Key:**
    -   Required for translation services.
    -   Obtain from Google AI Studio or similar.

---

## Usage

-   **Local Development:**
    1.  Set up your environment variables (you can use a `.env` file).
    2.  Install dependencies:
        ```bash
        pip install -r requirements.txt
        ```
    3.  Run the server:
        ```bash
        python main.py
        ```

-   **Deployment on Digital Ocean App Platform:**
    1.  Configure environment variables in the App Platform settings.
    2.  Set up HTTP routes, health checks, and commands as described in the [Digital Ocean App Platform Configuration](#digital-ocean-app-platform-configuration) section.
    3.  Deploy the application; the Run Command will trigger the start command.

---

## Error Handling and Logging

-   **Error Logging:**
    -   Logs detailed debug and error messages for:
        -   WebSocket connection issues.
        -   Audio processing errors.
        -   Transcription and translation failures.
        -   Rate limiting events.

-   **Transcription and Translation Logging:**
    -   Transcription results and events sent to Genesys are logged at the `INFO` level.
    -   Translation failures are logged with details.

-   **Graceful Shutdown:**
    -   Handles close transactions by sending a `"closed"` message to Genesys Cloud.
    -   Cleans up session resources (stops transcription threads, cancels tasks).

-   **Rate Limiting:**
    -   Implements backoff for `429` errors (rate limit exceeded) from Genesys.
    -   Supports retry-after durations from Genesys or HTTP headers.

---

## Configuration

All configurable parameters are defined in `config.py` and loaded from environment variables. Below is a list of required environment variables:

| Variable                        | Description                                                                   | Default                 |
| :------------------------------ | :---------------------------------------------------------------------------- | :---------------------- |
| `GOOGLE_CLOUD_PROJECT`          | Google Cloud project ID for Speech-to-Text API (required for Google provider) | -                       |
| `GOOGLE_APPLICATION_CREDENTIALS`| JSON key for Google Cloud service account (required for Google provider)      | -                       |
| `GOOGLE_SPEECH_MODEL`           | Google Speech recognition model ('`chirp_2`' or '`chirp`')                      | `chirp_2`               |
| `GOOGLE_TRANSLATION_MODEL`      | Google Gemini model for translation                                           | -                       |
| `GEMINI_API_KEY`                | API key for Google Gemini                                                     | -                       |
| `OPENAI_API_KEY`                | API key for OpenAI (required for OpenAI provider)                             | -                       |
| `OPENAI_SPEECH_MODEL`           | OpenAI Speech-to-Text model                                                   | `gpt-4o-mini-transcribe`|
| `DEFAULT_SPEECH_PROVIDER`       | Default speech provider if not specified in customConfig ('google' or 'openai') | `google`                |
| `GENESYS_API_KEY`               | API key for Genesys Cloud Transcription Connector                             | -                       |
| `GENESYS_ORG_ID`                | Genesys Cloud organization ID                                                 | -                       |
| `DEBUG`                         | Set to `"true"` for increased logging granularity                             | `false`                 |
| `SUPPORTED_LANGUAGES`           | Comma-separated list of supported input languages (e.g., "es-ES,it-IT,en-US") | `es-ES,it-IT`           |

---

## Known Issues

-   **Google Transcription:** Random numbers in the transcription:
    -   From time to time some arbitrary numbers show up in the transcription, totally unrelated to the conversation itself. It requires further investigation.

-   **OpenAI Transcription:** Synthetic word timing:
    -   Because OpenAI's Speech API doesn't provide word-level timing information, the connector generates synthetic timestamps based on transcript length.
    -   This may cause slight misalignment in the Genesys UI compared to actual speech timing.

-   **OpenAI Transcription:** Latency during speech detection:
    -   The utterance detection system waits for silence (`800ms`) to determine the end of speech.
    -   This introduces a small latency between someone speaking and the transcript appearing.
    -   This is a tradeoff to ensure complete utterances are captured rather than partial fragments.
