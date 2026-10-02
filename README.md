# AI-ApplicGitHub Project Description

# 🚀 AI-Powered Creator & Intelligence Workspace

A local-first AI application designed to bring **AI reasoning, workflow automation, YouTube intelligence, voice interaction, transcript analysis, and content creation** into one unified workspace.

## ✨ What does it do?

The application combines:

* 🤖 Local AI with Ollama
* 🎙️ Voice input
* ▶️ YouTube search and video intelligence
* 📝 Transcript processing
* 🧠 AI-assisted analysis
* ✍️ Content generation
* 🔄 Workflow-based execution
* 📊 Information intelligence
* 💾 Project and activity management

### 🔥 Core Workflow

```text
User Question
      ↓
AI Understanding
      ↓
Planning
      ↓
Related YouTube Content
      ↓
Video / Transcript
      ↓
AI Analysis
      ↓
Content Creation
      ↓
Final Output
```

## 🛠️ Technology

* Node.js
* Express.js
* JavaScript
* HTML / CSS
* Ollama
* Local AI
* YouTube integration
* Browser Speech Recognition
* SQLite

## 💡 Why I built it

I wanted to experiment with building an AI application that behaves more like a **real productivity and intelligence system** rather than simply a chat interface.

The project is continuously evolving.

## 🙏 I Need Your Feedback

If you try this project, please give me an honest review.

Tell me:

* What works well?
* What should be improved?
* Did you find any bugs?
* What would you change in the UI?
* What feature should I build next?
* What architectural improvements would you suggest?

⭐ If you find the project useful or interesting, please consider starring the repository.

💬 **Your feedback and comments are highly appreciated.**

server.cjs

"use strict";

/*
===========================================================
 CREATOROS
 LOCAL AI CREATOR OPERATING SYSTEM

 Stack:
 - Node.js
 - Express
 - SQLite
 - Ollama
 - YouTube Data API
 - youtube-transcript

 Local AI:
 http://127.0.0.1:11434
 Default model:
 phi3:latest

 .env example:

 PORT=3000
 OLLAMA_URL=http://127.0.0.1:11434
 MODEL_NAME=phi3:latest
 OLLAMA_TIMEOUT_MS=180000
 YT_API_KEY=YOUR_YOUTUBE_DATA_API_KEY
===========================================================
*/

require("dotenv").config();

const express = require("express");
const path = require("path");
const fs = require("fs");

const { DatabaseSync } = require("node:sqlite");

const app = express();

const PORT = Number(
  process.env.PORT || 3000
);

const OLLAMA_URL = String(
  process.env.OLLAMA_URL ||
    "http://127.0.0.1:11434"
).replace(/\/+$/, "");

const MODEL_NAME =
  process.env.MODEL_NAME ||
  process.env.OLLAMA_MODEL ||
  "phi3:latest";

const OLLAMA_TIMEOUT_MS = Number(
  process.env.OLLAMA_TIMEOUT_MS ||
    180000
);

const YT_API_KEY =
  process.env.YT_API_KEY || "";

const ROOT =
  __dirname;

const PUBLIC_DIR =
  path.join(
    ROOT,
    "public"
  );

const DATA_DIR =
  path.join(
    ROOT,
    "data"
  );

if (
  !fs.existsSync(
    DATA_DIR
  )
) {
  fs.mkdirSync(
    DATA_DIR,
    {
      recursive: true
    }
  );
}

if (
  !fs.existsSync(
    PUBLIC_DIR
  )
) {
  fs.mkdirSync(
    PUBLIC_DIR,
    {
      recursive: true
    }
  );
}

/*
===========================================================
 EXPRESS
===========================================================
*/

app.disable(
  "x-powered-by"
);

app.use(
  express.json({
    limit: "5mb"
  })
);

app.use(
  express.urlencoded({
    extended: true,
    limit: "5mb"
  })
);

app.use(
  express.static(
    PUBLIC_DIR,
    {
      extensions: [
        "html"
      ],
      maxAge: 0
    }
  )
);

/*
===========================================================
 DATABASE
===========================================================
*/

const dbPath =
  path.join(
    DATA_DIR,
    "creator-os.db"
  );

const db =
  new DatabaseSync(
    dbPath
  );

db.exec(`
  PRAGMA journal_mode = WAL;
  PRAGMA foreign_keys = ON;

  CREATE TABLE IF NOT EXISTS playlist (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    video_id TEXT NOT NULL UNIQUE,
    title TEXT,
    channel TEXT,
    thumbnail TEXT,
    duration TEXT,
    views INTEGER DEFAULT 0,
    added_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS memory (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    text TEXT NOT NULL,
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS activity (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,
    data TEXT,
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS notifications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    query TEXT NOT NULL,
    type TEXT DEFAULT 'channel',
    active INTEGER DEFAULT 1,
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS search_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    query TEXT NOT NULL,
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS creator_projects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    topic TEXT,
    format TEXT,
    content TEXT,
    created_at TEXT NOT NULL
  );
`);

/*
===========================================================
 HELPERS
===========================================================
*/

function now() {
  return new Date().toISOString();
}

function cleanText(
  value,
  max = 20000
) {
  return String(
    value || ""
  )
    .replace(
      /\u0000/g,
      ""
    )
    .slice(
      0,
      max
    );
}

function safeJson(
  value
) {
  try {
    return JSON.stringify(
      value
    );
  } catch {
    return "{}";
  }
}

function parseJson(
  value,
  fallback = {}
) {
  try {
    return JSON.parse(
      value
    );
  } catch {
    return fallback;
  }
}

function normalizeVideo(
  video = {}
) {
  const id =
    video.id ||
    video.videoId ||
    "";

  return {
    id: String(
      id
    ),

    title:
      cleanText(
        video.title ||
          video.name ||
          "Untitled video",
        500
      ),

    channel:
      cleanText(
        video.channel ||
          video.channelTitle ||
          "",
        300
      ),

    thumbnail:
      video.thumbnail ||
      `https://i.ytimg.com/vi/${encodeURIComponent(
        id
      )}/hqdefault.jpg`,

    duration:
      video.duration ||
      "",

    views:
      Number(
        video.views || 0
      ),

    likes:
      Number(
        video.likes || 0
      ),

    publishedAt:
      video.publishedAt ||
      ""
  };
}

/*
===========================================================
 ACTIVITY
===========================================================
*/

function addActivity(
  type,
  data = {}
) {
  try {
    db.prepare(
      `
      INSERT INTO activity
      (
        type,
        data,
        created_at
      )
      VALUES (?, ?, ?)
      `
    ).run(
      type,
      safeJson(data),
      now()
    );
  } catch {}
}

/*
===========================================================
 OLLAMA
===========================================================
*/

async function ollamaFetch(
  endpoint,
  options = {}
) {
  const controller =
    new AbortController();

  const timeout =
    setTimeout(
      () =>
        controller.abort(),
      options.timeout ||
        OLLAMA_TIMEOUT_MS
    );

  try {
    return await fetch(
      `${OLLAMA_URL}${endpoint}`,
      {
        ...options,
        signal:
          controller.signal,

        headers: {
          Accept:
            "application/json",

          ...(options.headers ||
            {})
        }
      }
    );
  } finally {
    clearTimeout(
      timeout
    );
  }
}

async function getOllamaModels() {
  const response =
    await ollamaFetch(
      "/api/tags",
      {
        method: "GET",
        timeout: 10000
      }
    );

  if (
    !response.ok
  ) {
    throw new Error(
      `Ollama returned HTTP ${response.status}`
    );
  }

  const data =
    await response.json();

  return (
    data.models || []
  );
}

async function checkOllama() {
  try {
    const models =
      await getOllamaModels();

    const names =
      models.map(
        model =>
          model.name
      );

    return {
      online: true,
      model:
        MODEL_NAME,
      available:
        names.includes(
          MODEL_NAME
        ),
      models:
        names
    };
  } catch (
    error
  ) {
    return {
      online: false,
      model:
        MODEL_NAME,
      available: false,
      models: [],
      error:
        error.name ===
        "AbortError"
          ? "Ollama timeout"
          : error.message
    };
  }
}

/*
===========================================================
 SSE
===========================================================
*/

function setupSSE(
  res
) {
  res.status(200);

  res.setHeader(
    "Content-Type",
    "text/event-stream; charset=utf-8"
  );

  res.setHeader(
    "Cache-Control",
    "no-cache, no-store, must-revalidate"
  );

  res.setHeader(
    "Connection",
    "keep-alive"
  );

  res.setHeader(
    "X-Accel-Buffering",
    "no"
  );

  res.setHeader(
    "Content-Encoding",
    "identity"
  );

  if (
    typeof res.flushHeaders ===
    "function"
  ) {
    res.flushHeaders();
  }

  const heartbeat =
    setInterval(
      () => {
        if (
          !res.writableEnded
        ) {
          res.write(
            `: heartbeat ${Date.now()}\n\n`
          );
        }
      },
      15000
    );

  return {
    send(
      event,
      payload
    ) {
      if (
        res.writableEnded
      ) {
        return;
      }

      res.write(
        `event: ${event}\n`
      );

      res.write(
        `data: ${JSON.stringify(
          payload
        )}\n\n`
      );
    },

    close() {
      clearInterval(
        heartbeat
      );

      if (
        !res.writableEnded
      ) {
        res.end();
      }
    }
  };
}

async function streamOllamaToSSE({
  res,
  prompt,
  system,
  temperature = 0.25,
  numCtx = 4096
}) {
  const sse =
    setupSSE(
      res
    );

  const controller =
    new AbortController();

  let closed =
    false;

  const onClose =
    () => {
      closed = true;

      try {
        controller.abort();
      } catch {}
    };

  res.on(
    "close",
    onClose
  );

  try {
    sse.send(
      "ai_start",
      {
        model:
          MODEL_NAME
      }
    );

    const response =
      await fetch(
        `${OLLAMA_URL}/api/generate`,
        {
          method:
            "POST",

          headers: {
            "Content-Type":
              "application/json",

            Accept:
              "application/x-ndjson"
          },

          body:
            JSON.stringify({
              model:
                MODEL_NAME,

              prompt:
                cleanText(
                  prompt,
                  40000
                ),

              system:
                cleanText(
                  system ||
                    "You are CreatorOS, a practical local AI creator assistant.",
                  15000
                ),

              stream:
                true,

              options: {
                temperature,
                num_ctx:
                  numCtx
              }
            }),

          signal:
            controller.signal
        }
      );

    if (
      !response.ok
    ) {
      const errorText =
        await response.text();

      throw new Error(
        `Ollama HTTP ${response.status}: ${errorText.slice(
          0,
          700
        )}`
      );
    }

    if (
      !response.body
    ) {
      throw new Error(
        "Ollama returned no streaming body."
      );
    }

    const reader =
      response.body.getReader();

    const decoder =
      new TextDecoder();

    let buffer =
      "";

    while (
      !closed
    ) {
      const {
        value,
        done
      } =
        await reader.read();

      if (
        done
      ) {
        break;
      }

      buffer +=
        decoder.decode(
          value,
          {
            stream: true
          }
        );

      const lines =
        buffer.split(
          "\n"
        );

      buffer =
        lines.pop() ||
        "";

      for (
        const line of lines
      ) {
        const trimmed =
          line.trim();

        if (
          !trimmed
        ) {
          continue;
        }

        let chunk;

        try {
          chunk =
            JSON.parse(
              trimmed
            );
        } catch {
          continue;
        }

        if (
          chunk.response
        ) {
          sse.send(
            "ai_chunk",
            {
              text:
                chunk.response
            }
          );
        }

        if (
          chunk.done
        ) {
          sse.send(
            "ai_done",
            {
              model:
                chunk.model ||
                MODEL_NAME,

              total_duration:
                chunk.total_duration ||
                0
            }
          );
        }
      }
    }

    if (
      !closed
    ) {
      sse.send(
        "ai_complete",
        {
          ok: true
        }
      );
    }
  } catch (
    error
  ) {
    if (
      !closed
    ) {
      sse.send(
        "ai_error",
        {
          error:
            error.name ===
            "AbortError"
              ? "Ollama request cancelled or timed out."
              : error.message
        }
      );
    }
  } finally {
    res.off(
      "close",
      onClose
    );

    sse.close();
  }
}

/*
===========================================================
 YOUTUBE API
===========================================================
*/

async function youtubeRequest(
  endpoint,
  params = {}
) {
  if (
    !YT_API_KEY
  ) {
    throw new Error(
      "YT_API_KEY is missing in .env."
    );
  }

  const url =
    new URL(
      `https://www.googleapis.com/youtube/v3/${endpoint}`
    );

  Object.entries({
    ...params,
    key:
      YT_API_KEY
  }).forEach(
    ([
      key,
      value
    ]) => {
      if (
        value !==
          undefined &&
        value !== null
      ) {
        url.searchParams.set(
          key,
          value
        );
      }
    }
  );

  const response =
    await fetch(
      url,
      {
        headers: {
          Accept:
            "application/json"
        }
      }
    );

  const data =
    await response.json();

  if (
    !response.ok
  ) {
    throw new Error(
      data?.error
        ?.message ||
        `YouTube API HTTP ${response.status}`
    );
  }

  return data;
}

function isoDurationToText(
  iso
) {
  if (
    !iso
  ) {
    return "";
  }

  const match =
    iso.match(
      /PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?/
    );

  if (
    !match
  ) {
    return "";
  }

  const hours =
    Number(
      match[1] || 0
    );

  const minutes =
    Number(
      match[2] || 0
    );

  const seconds =
    Number(
      match[3] || 0
    );

  if (
    hours
  ) {
    return `${hours}:${String(
      minutes
    ).padStart(
      2,
      "0"
    )}:${String(
      seconds
    ).padStart(
      2,
      "0"
    )}`;
  }

  return `${minutes}:${String(
    seconds
  ).padStart(
    2,
    "0"
  )}`;
}

async function searchYouTube(
  query,
  limit = 12
) {
  const search =
    await youtubeRequest(
      "search",
      {
        part:
          "snippet",

        q:
          query,

        type:
          "video",

        maxResults:
          Math.min(
            Math.max(
              Number(
                limit
              ) || 12,
              1
            ),
            50
          )
      }
    );

  const ids =
    (
      search.items ||
      []
    )
      .map(
        item =>
          item.id
            ?.videoId
      )
      .filter(Boolean);

  if (
    !ids.length
  ) {
    return [];
  }

  const details =
    await youtubeRequest(
      "videos",
      {
        part:
          "snippet,contentDetails,statistics",

        id:
          ids.join(",")
      }
    );

  const detailMap =
    new Map(
      (
        details.items ||
        []
      ).map(
        item => [
          item.id,
          item
        ]
      )
    );

  return ids.map(
    id => {
      const detail =
        detailMap.get(
          id
        );

      const snippet =
        detail?.snippet ||
        (
          search.items ||
          []
        ).find(
          item =>
            item.id
              ?.videoId ===
            id
        )?.snippet ||
        {};

      const stats =
        detail?.statistics ||
        {};

      return normalizeVideo({
        id,

        title:
          snippet.title,

        channel:
          snippet.channelTitle,

        thumbnail:
          snippet.thumbnails
            ?.high?.url ||
          snippet.thumbnails
            ?.medium?.url ||
          snippet.thumbnails
            ?.default?.url,

        duration:
          isoDurationToText(
            detail
              ?.contentDetails
              ?.duration
          ),

        views:
          Number(
            stats.viewCount ||
              0
          ),

        likes:
          Number(
            stats.likeCount ||
              0
          ),

        publishedAt:
          snippet.publishedAt
      });
    }
  );
}

/*
===========================================================
 YOUTUBE SEARCH
===========================================================
*/

app.get(
  "/api/youtube/search",
  async (
    req,
    res
  ) => {
    try {
      const query =
        cleanText(
          req.query.q,
          500
        ).trim();

      const limit =
        Number(
          req.query.limit ||
            12
        );

      if (
        !query
      ) {
        return res
          .status(400)
          .json({
            error:
              "Search query is required."
          });
      }

      const videos =
        await searchYouTube(
          query,
          limit
        );

      db.prepare(
        `
        INSERT INTO search_history
        (
          query,
          created_at
        )
        VALUES (?, ?)
        `
      ).run(
        query,
        now()
      );

      addActivity(
        "youtube_search",
        {
          query,
          count:
            videos.length
        }
      );

      res.json({
        query,
        videos
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 YOUTUBE METADATA
===========================================================
*/

app.get(
  "/api/youtube/metadata/:id",
  async (
    req,
    res
  ) => {
    try {
      const id =
        cleanText(
          req.params.id,
          100
        ).trim();

      if (
        !id
      ) {
        return res
          .status(400)
          .json({
            error:
              "Video ID is required."
          });
      }

      const data =
        await youtubeRequest(
          "videos",
          {
            part:
              "snippet,contentDetails,statistics,status",

            id
          }
        );

      const item =
        data.items?.[0];

      if (
        !item
      ) {
        return res
          .status(404)
          .json({
            error:
              "Video not found."
          });
      }

      res.json({
        ...normalizeVideo({
          id:
            item.id,

          title:
            item.snippet
              ?.title,

          channel:
            item.snippet
              ?.channelTitle,

          thumbnail:
            item.snippet
              ?.thumbnails
              ?.high?.url,

          duration:
            isoDurationToText(
              item.contentDetails
                ?.duration
            ),

          views:
            Number(
              item.statistics
                ?.viewCount ||
                0
            ),

          likes:
            Number(
              item.statistics
                ?.likeCount ||
                0
            ),

          publishedAt:
            item.snippet
              ?.publishedAt
        }),

        description:
          item.snippet
            ?.description ||
          "",

        tags:
          item.snippet?.tags ||
          [],

        categoryId:
          item.snippet
            ?.categoryId ||
          "",

        privacyStatus:
          item.status
            ?.privacyStatus ||
          ""
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 TRANSCRIPT HELPER
===========================================================
*/

async function getTranscriptText(
  videoId
) {
  let transcriptModule;

  try {
    transcriptModule =
      require(
        "youtube-transcript"
      );
  } catch {
    throw new Error(
      "youtube-transcript is not installed. Run: npm install youtube-transcript"
    );
  }

  const Transcript =
    transcriptModule.Transcript ||
    transcriptModule;

  let items =
    [];

  if (
    typeof Transcript.getTranscript ===
    "function"
  ) {
    items =
      await Transcript.getTranscript(
        videoId
      );
  } else if (
    typeof Transcript.fetchTranscript ===
    "function"
  ) {
    items =
      await Transcript.fetchTranscript(
        videoId
      );
  } else if (
    typeof transcriptModule.getTranscript ===
    "function"
  ) {
    items =
      await transcriptModule.getTranscript(
        videoId
      );
  } else {
    throw new Error(
      "Unsupported youtube-transcript package API."
    );
  }

  const text =
    (
      items ||
      []
    )
      .map(
        item =>
          item.text ||
          item.snippet ||
          ""
      )
      .join(" ")
      .replace(
        /\s+/g,
        " "
      )
      .trim();

  if (
    !text
  ) {
    throw new Error(
      "No transcript was available for this video."
    );
  }

  return text;
}

/*
===========================================================
 YOUTUBE CONTEXT
 Metadata + Transcript
===========================================================
*/

app.get(
  "/api/youtube/context/:id",
  async (
    req,
    res
  ) => {
    try {
      const id =
        cleanText(
          req.params.id,
          100
        ).trim();

      if (
        !id
      ) {
        return res
          .status(400)
          .json({
            error:
              "Video ID is required."
          });
      }

      const data =
        await youtubeRequest(
          "videos",
          {
            part:
              "snippet,contentDetails,statistics,status",

            id
          }
        );

      const item =
        data.items?.[0];

      if (
        !item
      ) {
        return res
          .status(404)
          .json({
            error:
              "Video not found."
          });
      }

      const video = {
        ...normalizeVideo({
          id:
            item.id,

          title:
            item.snippet
              ?.title,

          channel:
            item.snippet
              ?.channelTitle,

          thumbnail:
            item.snippet
              ?.thumbnails
              ?.high?.url ||
            item.snippet
              ?.thumbnails
              ?.medium?.url,

          duration:
            isoDurationToText(
              item.contentDetails
                ?.duration
            ),

          views:
            Number(
              item.statistics
                ?.viewCount ||
                0
            ),

          likes:
            Number(
              item.statistics
                ?.likeCount ||
                0
            ),

          publishedAt:
            item.snippet
              ?.publishedAt
        }),

        description:
          item.snippet
            ?.description ||
          "",

        tags:
          item.snippet?.tags ||
          [],

        categoryId:
          item.snippet
            ?.categoryId ||
          "",

        privacyStatus:
          item.status
            ?.privacyStatus ||
          ""
      };

      let transcript =
        "";

      let transcriptAvailable =
        false;

      let transcriptError =
        "";

      try {
        transcript =
          await getTranscriptText(
            id
          );

        transcriptAvailable =
          Boolean(
            transcript
          );
      } catch (
        error
      ) {
        transcriptError =
          error.message;
      }

      addActivity(
        "youtube_context",
        {
          videoId:
            id,

          transcript:
            transcriptAvailable
        }
      );

      res.json({
        video,

        transcript,

        transcriptAvailable,

        transcriptError
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 TRANSCRIPT ONLY
===========================================================
*/

app.get(
  "/api/youtube/transcript/:id",
  async (
    req,
    res
  ) => {
    try {
      const id =
        cleanText(
          req.params.id,
          100
        ).trim();

      const text =
        await getTranscriptText(
          id
        );

      addActivity(
        "transcript_loaded",
        {
          videoId:
            id,

          characters:
            text.length
        }
      );

      res.json({
        videoId:
          id,

        text,

        length:
          text.length
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 QUESTION → YOUTUBE RECOMMENDATIONS
===========================================================
*/

app.post(
  "/api/youtube/recommend",
  async (
    req,
    res
  ) => {
    try {
      const question =
        cleanText(
          req.body?.question,
          5000
        ).trim();

      if (
        !question
      ) {
        return res
          .status(400)
          .json({
            error:
              "Question is required."
          });
      }

      let youtubeQuery =
        question;

      /*
      Use Ollama only to turn the
      user's question into a clean
      YouTube search phrase.

      Deterministic YouTube API search
      then retrieves the actual videos.
      */

      try {
        const response =
          await fetch(
            `${OLLAMA_URL}/api/generate`,
            {
              method:
                "POST",

              headers: {
                "Content-Type":
                  "application/json"
              },

              body:
                JSON.stringify({
                  model:
                    MODEL_NAME,

                  prompt: `
Convert this user question into one concise YouTube search query.

User question:
${question}

Return ONLY the search query.
No explanation.
No quotes.
`,

                  stream:
                    false,

                  options: {
                    temperature:
                      0.1,

                    num_ctx:
                      1024
                  }
                })
            }
          );

        if (
          response.ok
        ) {
          const result =
            await response.json();

          youtubeQuery =
            cleanText(
              result.response,
              500
            )
              .replace(
                /^["']|["']$/g,
                ""
              )
              .trim() ||
            question;
        }
      } catch {
        youtubeQuery =
          question;
      }

      const videos =
        await searchYouTube(
          youtubeQuery,
          8
        );

      addActivity(
        "question_youtube",
        {
          question,
          youtubeQuery,
          count:
            videos.length
        }
      );

      res.json({
        question,

        query:
          youtubeQuery,

        videos
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 RECOMMENDATIONS COMPATIBILITY ROUTE
===========================================================
*/

app.post(
  "/api/recommendations",
  async (
    req,
    res
  ) => {
    try {
      const topic =
        cleanText(
          req.body?.topic,
          5000
        ).trim();

      if (
        !topic
      ) {
        return res.json({
          videos: []
        });
      }

      const videos =
        await searchYouTube(
          topic,
          6
        );

      res.json({
        query:
          topic,

        videos
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 VIDEO AI
===========================================================
*/

function buildVideoPrompt({
  action,
  video,
  transcript
}) {
  const actions = {
    summary:
      `
Summarize the supplied video transcript.
Give:
1. Executive summary
2. Main points
3. Important details
4. Key takeaway
`,

    analysis:
      `
Perform deep content analysis.
Cover:
1. Hook
2. Structure
3. Audience
4. Retention beats
5. Main arguments
6. Examples
7. Creator techniques
8. Opportunities for improvement
`,

    notes:
      `
Convert the transcript into structured research notes.
Use clear headings and concise bullet points.
`,

    quiz:
      `
Create a useful quiz based ONLY on the transcript.
Include questions and answers.
`,

    shorts:
      `
Create 5 original YouTube Shorts concepts from
the source transcript.

For each:
- Hook
- Short script
- Visual idea
- CTA
- Title
`,

    title:
      `
Generate 15 original YouTube title ideas based
strictly on the supplied subject matter.
`,

    hashtags:
      `
Generate relevant YouTube hashtags based on the
actual content.
`,

    linkedin:
      `
Create a professional LinkedIn post based on
the useful ideas in the transcript.
Do not invent facts.
`
  };

  return `
VIDEO:

Title:
${video.title}

Channel:
${video.channel}

Duration:
${video.duration}

Views:
${video.views}

Transcript:
${transcript}

TASK:
${
  actions[action] ||
  actions.summary
}

RULES:
- Do not fabricate facts.
- Do not invent transcript statements.
- Clearly label creative suggestions.
- Do not pretend to have browsed external sources.
- Keep the output useful and structured.
`;
}

app.post(
  "/api/youtube/ai",
  async (
    req,
    res
  ) => {
    const action =
      cleanText(
        req.body?.action,
        100
      ).trim();

    const video =
      normalizeVideo(
        req.body?.video ||
          {}
      );

    const transcript =
      cleanText(
        req.body?.transcript,
        60000
      );

    if (
      !video.id
    ) {
      return res
        .status(400)
        .json({
          error:
            "Select a video first."
        });
    }

    if (
      !transcript
    ) {
      return res
        .status(400)
        .json({
          error:
            "Load a transcript first."
        });
    }

    await streamOllamaToSSE({
      res,

      prompt:
        buildVideoPrompt({
          action,
          video,
          transcript
        }),

      system: `
You are CreatorOS Video Intelligence.

You analyze only the supplied YouTube
metadata and transcript.

Do not fabricate external information.
Do not pretend to browse.
`,

      temperature:
        0.2,

      numCtx:
        8192
    });
  }
);

/*
===========================================================
 CREATOROS MAIN QUESTION
===========================================================
*/

app.post(
  "/api/creator/stream",
  async (
    req,
    res
  ) => {
    const question =
      cleanText(
        req.body?.question,
        6000
      ).trim();

    const mission =
      cleanText(
        req.body?.mission,
        6000
      ).trim();

    const input =
      question ||
      mission;

    if (
      !input
    ) {
      return res
        .status(400)
        .json({
          error:
            "Ask CreatorOS something first."
        });
    }

    const memory =
      db
        .prepare(
          `
          SELECT text
          FROM memory
          ORDER BY id DESC
          LIMIT 8
          `
        )
        .all()
        .map(
          item =>
            `- ${item.text}`
        )
        .join("\n");

    const playlist =
      db
        .prepare(
          `
          SELECT
            title,
            channel
          FROM playlist
          ORDER BY id DESC
          LIMIT 8
          `
        )
        .all()
        .map(
          item =>
            `- ${item.title} — ${item.channel}`
        )
        .join("\n");

    addActivity(
      "creator_question",
      {
        question:
          input
      }
    );

    const prompt = `
USER REQUEST:
${input}

LOCAL CREATOROS MEMORY:
${
  memory ||
  "No saved memory."
}

CURRENT PLAYLIST:
${
  playlist ||
  "Playlist empty."
}

You are CreatorOS.

Answer the user's question directly.

Rules:
- Give a useful, practical answer.
- Think like a creator strategist.
- Provide structure where useful.
- Give examples when helpful.
- Do not invent external facts.
- Do not claim you browsed the internet.
- Do not claim you watched a YouTube video unless its transcript was supplied.
- You are running locally through Ollama.
`;

    await streamOllamaToSSE({
      res,

      prompt,

      system: `
You are CreatorOS, a local-first AI creator
operating system.

You help with:
- content ideas
- scripts
- YouTube
- Shorts
- long videos
- research
- creator strategy
- repurposing
- production planning

Always provide practical answers.
`,

      temperature:
        0.35,

      numCtx:
        4096
    });
  }
);

/*
===========================================================
 CREATOR FORMATS
===========================================================
*/

const CREATOR_FORMATS = [
  {
    id:
      "short",

    icon:
      "⚡",

    label:
      "YouTube Short",

    description:
      "15–60 sec focused short",

    duration:
      "15–60 sec"
  },

  {
    id:
      "long",

    icon:
      "▣",

    label:
      "Long-form Video",

    description:
      "6–20 min structured video",

    duration:
      "6–20 min"
  },

  {
    id:
      "advanced",

    icon:
      "◈",

    label:
      "Advanced Video",

    description:
      "Deep cinematic production",

    duration:
      "8–30 min"
  },

  {
    id:
      "shorts_pack",

    icon:
      "✦",

    label:
      "Shorts Pack",

    description:
      "3–10 connected Shorts",

    duration:
      "15–60 sec each"
  },

  {
    id:
      "long_to_shorts",

    icon:
      "↗",

    label:
      "Long → Shorts",

    description:
      "Repurpose long script/transcript",

    duration:
      "15–60 sec each"
  },

  {
    id:
      "story",

    icon:
      "♡",

    label:
      "Story / Emotional",

    description:
      "Narrative-first creator story",

    duration:
      "1–10 min"
  },

  {
    id:
      "educational",

    icon:
      "⌘",

    label:
      "Educational",

    description:
      "Teach clearly with structure",

    duration:
      "2–20 min"
  },

  {
    id:
      "devotional",

    icon:
      "✧",

    label:
      "Devotional",

    description:
      "Respectful devotional content",

    duration:
      "30 sec–10 min"
  }
];

function creatorFormatById(
  id
) {
  return (
    CREATOR_FORMATS.find(
      item =>
        item.id === id
    ) ||
    CREATOR_FORMATS[0]
  );
}

function creatorGenerationPrompt({
  topic,
  sourceText,
  format,
  language,
  style,
  duration,
  shortsCount
}) {
  const selected =
    creatorFormatById(
      format
    );

  const count =
    Math.min(
      Math.max(
        Number(
          shortsCount
        ) || 5,
        3
      ),
      10
    );

  return `
You are CreatorOS Creator Studio.

TOPIC:
${topic || "Use source material."}

FORMAT:
${selected.label}

DESCRIPTION:
${selected.description}

LANGUAGE:
${language || "English"}

STYLE:
${style || "Modern & engaging"}

TARGET DURATION:
${duration || selected.duration}

SOURCE / TRANSCRIPT:
${sourceText || "No source transcript supplied."}

Create original production-ready content.

Rules:
- Do not copy another creator.
- Do not fabricate facts.
- Preserve source meaning.
- Use strong hooks.
- Give useful visual directions.
- Include CTA.
- Include title ideas.
- Include description.
- Include hashtags.

${
  format ===
  "short"
    ? `
Create one complete YouTube Short:
Hook
Script
On-screen text
Visual/B-roll
CTA
Title
Description
Hashtags
`
    : ""
}

${
  format ===
  "long"
    ? `
Create a complete long-form video:
Cold open
Introduction
Chapters
Spoken script
B-roll
Visual direction
Transitions
Retention beats
CTA
Titles
Description
Hashtags
`
    : ""
}

${
  format ===
  "advanced"
    ? `
Create an advanced production blueprint:
Opening sequence
Narrative arc
Scene-by-scene script
Visual direction
B-roll
Sound cues
Retention beats
CTA
Titles
Metadata
`
    : ""
}

${
  format ===
  "shorts_pack" ||
  format ===
  "long_to_shorts"
    ? `
Create ${count} distinct Shorts.

For each:
1. Short title
2. Hook
3. Script
4. On-screen text
5. Visual/B-roll
6. CTA
7. Hashtags

Each Short should feel different.
`
    : ""
}

${
  format ===
  "story"
    ? `
Create an emotional narrative:
Opening
Character/context
Conflict
Emotional progression
Resolution
Voiceover
Visual direction
CTA
`
    : ""
}

${
  format ===
  "educational"
    ? `
Create:
Hook
Learning objectives
Step-by-step explanation
Examples
Common mistakes
Recap
CTA
`
    : ""
}

${
  format ===
  "devotional"
    ? `
Create respectful devotional content.
Use original wording.
Do not fabricate scripture quotations.
Do not claim unsupported religious facts.
`
    : ""
}

Return only creator-ready content.
`;
}

app.get(
  "/api/creator/formats",
  (
    req,
    res
  ) => {
    res.json({
      formats:
        CREATOR_FORMATS
    });
  }
);

app.post(
  "/api/creator/generate",
  async (
    req,
    res
  ) => {
    try {
      const payload = {
        topic:
          cleanText(
            req.body?.topic,
            10000
          ).trim(),

        sourceText:
          cleanText(
            req.body?.sourceText,
            30000
          ).trim(),

        format:
          cleanText(
            req.body?.format,
            100
          ).trim(),

        language:
          cleanText(
            req.body?.language,
            100
          ).trim(),

        style:
          cleanText(
            req.body?.style,
            200
          ).trim(),

        duration:
          cleanText(
            req.body?.duration,
            100
          ).trim(),

        shortsCount:
          Number(
            req.body?.shortsCount ||
              5
          )
      };

      if (
        !payload.topic &&
        !payload.sourceText
      ) {
        return res
          .status(400)
          .json({
            error:
              "Add a topic or source material first."
          });
      }

      const response =
        await fetch(
          `${OLLAMA_URL}/api/generate`,
          {
            method:
              "POST",

            headers: {
              "Content-Type":
                "application/json"
            },

            body:
              JSON.stringify({
                model:
                  MODEL_NAME,

                prompt:
                  creatorGenerationPrompt(
                    payload
                  ),

                stream:
                  false,

                options: {
                  temperature:
                    0.55,

                  num_ctx:
                    8192
                }
              })
          }
        );

      if (
        !response.ok
      ) {
        throw new Error(
          `Ollama HTTP ${response.status}`
        );
      }

      const result =
        await response.json();

      let relatedVideos =
        [];

      if (
        YT_API_KEY &&
        payload.topic
      ) {
        try {
          relatedVideos =
            await searchYouTube(
              payload.topic,
              6
            );
        } catch {}
      }

      addActivity(
        "creator_generation",
        {
          topic:
            payload.topic,

          format:
            payload.format
        }
      );

      res.json({
        ok:
          true,

        content:
          result.response ||
          "",

        format:
          creatorFormatById(
            payload.format
          ),

        relatedVideos
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

app.post(
  "/api/creator/generate-stream",
  async (
    req,
    res
  ) => {
    const payload = {
      topic:
        cleanText(
          req.body?.topic,
          10000
        ).trim(),

      sourceText:
        cleanText(
          req.body?.sourceText,
          30000
        ).trim(),

      format:
        cleanText(
          req.body?.format,
          100
        ).trim(),

      language:
        cleanText(
          req.body?.language,
          100
        ).trim(),

      style:
        cleanText(
          req.body?.style,
          200
        ).trim(),

      duration:
        cleanText(
          req.body?.duration,
          100
        ).trim(),

      shortsCount:
        Number(
          req.body?.shortsCount ||
            5
        )
    };

    if (
      !payload.topic &&
      !payload.sourceText
    ) {
      return res
        .status(400)
        .json({
          error:
            "Add a topic or source material first."
        });
    }

    addActivity(
      "creator_generation_stream",
      {
        topic:
          payload.topic,

        format:
          payload.format
      }
    );

    await streamOllamaToSSE({
      res,

      prompt:
        creatorGenerationPrompt(
          payload
        ),

      system:
        "You are CreatorOS Creator Studio. Produce original, practical, production-ready creator content.",

      temperature:
        0.55,

      numCtx:
        8192
    });
  }
);

/*
===========================================================
 CREATOR PROJECTS
===========================================================
*/

app.get(
  "/api/creator/projects",
  (
    req,
    res
  ) => {
    try {
      const projects =
        db
          .prepare(
            `
            SELECT
              id,
              title,
              topic,
              format,
              content,
              created_at AS createdAt
            FROM creator_projects
            ORDER BY id DESC
            LIMIT 100
            `
          )
          .all();

      res.json({
        projects
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

app.post(
  "/api/creator/projects",
  (
    req,
    res
  ) => {
    try {
      const title =
        cleanText(
          req.body?.title,
          200
        ).trim() ||
        "CreatorOS Project";

      const topic =
        cleanText(
          req.body?.topic,
          10000
        ).trim();

      const format =
        cleanText(
          req.body?.format,
          100
        ).trim() ||
        "short";

      const content =
        cleanText(
          req.body?.content,
          100000
        ).trim();

      if (
        !content
      ) {
        return res
          .status(400)
          .json({
            error:
              "Project content is required."
          });
      }

      const createdAt =
        now();

      const result =
        db.prepare(
          `
          INSERT INTO creator_projects
          (
            title,
            topic,
            format,
            content,
            created_at
          )
          VALUES (?, ?, ?, ?, ?)
          `
        ).run(
          title,
          topic,
          format,
          content,
          createdAt
        );

      addActivity(
        "creator_project_saved",
        {
          id:
            result.lastInsertRowid,

          title,

          format
        }
      );

      res.json({
        ok:
          true,

        id:
          result.lastInsertRowid,

        createdAt
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 PLAYLIST
===========================================================
*/

function getPlaylist() {
  return db
    .prepare(
      `
      SELECT
        id,
        video_id AS videoId,
        title,
        channel,
        thumbnail,
        duration,
        views,
        added_at AS addedAt
      FROM playlist
      ORDER BY id ASC
      `
    )
    .all()
    .map(
      item =>
        normalizeVideo({
          id:
            item.videoId,

          title:
            item.title,

          channel:
            item.channel,

          thumbnail:
            item.thumbnail,

          duration:
            item.duration,

          views:
            item.views,

          publishedAt:
            item.addedAt
        })
    );
}

app.get(
  "/api/playlist",
  (
    req,
    res
  ) => {
    res.json({
      playlist:
        getPlaylist()
    });
  }
);

app.post(
  "/api/playlist",
  (
    req,
    res
  ) => {
    try {
      const video =
        normalizeVideo(
          req.body?.video ||
            {}
        );

      if (
        !video.id
      ) {
        return res
          .status(400)
          .json({
            error:
              "Video ID is required."
          });
      }

      db.prepare(
        `
        INSERT OR IGNORE INTO playlist
        (
          video_id,
          title,
          channel,
          thumbnail,
          duration,
          views,
          added_at
        )
        VALUES (?, ?, ?, ?, ?, ?, ?)
        `
      ).run(
        video.id,
        video.title,
        video.channel,
        video.thumbnail,
        video.duration,
        video.views,
        now()
      );

      addActivity(
        "playlist_add",
        {
          videoId:
            video.id
        }
      );

      res.json({
        ok:
          true,

        playlist:
          getPlaylist()
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

app.post(
  "/api/playlist/clear",
  (
    req,
    res
  ) => {
    db.exec(
      "DELETE FROM playlist"
    );

    addActivity(
      "playlist_clear",
      {}
    );

    res.json({
      ok:
        true
    });
  }
);

/*
===========================================================
 MEMORY
===========================================================
*/

app.get(
  "/api/memory",
  (
    req,
    res
  ) => {
    const memory =
      db
        .prepare(
          `
          SELECT
            id,
            text,
            created_at AS createdAt
          FROM memory
          ORDER BY id DESC
          `
        )
        .all();

    res.json({
      memory
    });
  }
);

app.post(
  "/api/memory",
  (
    req,
    res
  ) => {
    try {
      const text =
        cleanText(
          req.body?.text,
          5000
        ).trim();

      if (
        !text
      ) {
        return res
          .status(400)
          .json({
            error:
              "Memory text is required."
          });
      }

      db.prepare(
        `
        INSERT INTO memory
        (
          text,
          created_at
        )
        VALUES (?, ?)
        `
      ).run(
        text,
        now()
      );

      addActivity(
        "memory_saved",
        {
          text
        }
      );

      res.json({
        ok:
          true
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 ACTIVITY
===========================================================
*/

app.get(
  "/api/activity",
  (
    req,
    res
  ) => {
    const rows =
      db
        .prepare(
          `
          SELECT
            id,
            type,
            data,
            created_at AS createdAt
          FROM activity
          ORDER BY id DESC
          LIMIT 100
          `
        )
        .all()
        .map(
          row => ({
            ...row,

            data:
              parseJson(
                row.data,
                {}
              )
          })
        );

    res.json({
      activity:
        rows
    });
  }
);

/*
===========================================================
 NOTIFICATIONS
===========================================================
*/

app.get(
  "/api/notifications",
  (
    req,
    res
  ) => {
    const notifications =
      db
        .prepare(
          `
          SELECT
            id,
            query,
            type,
            active,
            created_at AS createdAt
          FROM notifications
          ORDER BY id DESC
          `
        )
        .all();

    res.json({
      notifications
    });
  }
);

app.post(
  "/api/notifications",
  (
    req,
    res
  ) => {
    try {
      const query =
        cleanText(
          req.body?.query,
          500
        ).trim();

      const type =
        cleanText(
          req.body?.type,
          100
        ).trim() ||
        "channel";

      if (
        !query
      ) {
        return res
          .status(400)
          .json({
            error:
              "Notification query is required."
          });
      }

      db.prepare(
        `
        INSERT INTO notifications
        (
          query,
          type,
          active,
          created_at
        )
        VALUES (?, ?, 1, ?)
        `
      ).run(
        query,
        type,
        now()
      );

      addActivity(
        "notification_created",
        {
          query,
          type
        }
      );

      res.json({
        ok:
          true
      });
    } catch (
      error
    ) {
      res
        .status(500)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 SEARCH HISTORY
===========================================================
*/

app.get(
  "/api/search-history",
  (
    req,
    res
  ) => {
    const history =
      db
        .prepare(
          `
          SELECT
            id,
            query,
            created_at AS createdAt
          FROM search_history
          ORDER BY id DESC
          LIMIT 100
          `
        )
        .all();

    res.json({
      history
    });
  }
);

/*
===========================================================
 HEALTH
===========================================================
*/

app.get(
  "/api/health",
  async (
    req,
    res
  ) => {
    const ollama =
      await checkOllama();

    res.json({
      ok:
        true,

      app:
        "CreatorOS",

      server:
        true,

      model:
        MODEL_NAME,

      ollama,

      youtube:
        Boolean(
          YT_API_KEY
        ),

      timestamp:
        now()
    });
  }
);

/*
===========================================================
 OLLAMA MODELS
===========================================================
*/

app.get(
  "/api/ollama/models",
  async (
    req,
    res
  ) => {
    try {
      const models =
        await getOllamaModels();

      res.json({
        models:
          models.map(
            model =>
              model.name
          )
      });
    } catch (
      error
    ) {
      res
        .status(503)
        .json({
          error:
            error.message
        });
    }
  }
);

/*
===========================================================
 YOUTUBE COMMAND
===========================================================
*/

app.post(
  "/api/youtube/command",
  (
    req,
    res
  ) => {
    const command =
      cleanText(
        req.body?.command,
        2000
      ).trim();

    if (
      !command
    ) {
      return res
        .status(400)
        .json({
          error:
            "Command is required."
        });
    }

    const lower =
      command.toLowerCase();

    let action =
      "search";

    if (
      lower.includes(
        "play"
      )
    ) {
      action =
        "play";
    } else if (
      lower.includes(
        "pause"
      )
    ) {
      action =
        "pause";
    } else if (
      lower.includes(
        "next"
      )
    ) {
      action =
        "next";
    } else if (
      lower.includes(
        "previous"
      ) ||
      lower.includes(
        "prev"
      )
    ) {
      action =
        "previous";
    } else if (
      lower.includes(
        "restart"
      )
    ) {
      action =
        "restart";
    } else if (
      lower.includes(
        "transcript"
      )
    ) {
      action =
        "transcript";
    }

    res.json({
      action,
      command
    });
  }
);

/*
===========================================================
 ERROR HANDLER
===========================================================
*/

app.use(
  (
    error,
    req,
    res,
    next
  ) => {
    console.error(
      "[CreatorOS]",
      error
    );

    if (
      res.headersSent
    ) {
      return next(
        error
      );
    }

    res
      .status(500)
      .json({
        error:
          error.message ||
          "Internal server error."
      });
  }
);

/*
===========================================================
 FRONTEND FALLBACK
===========================================================
*/

app.use(
  (
    req,
    res,
    next
  ) => {
    if (
      req.method !==
        "GET" ||
      req.path.startsWith(
        "/api/"
      )
    ) {
      return next();
    }

    const indexPath =
      path.join(
        PUBLIC_DIR,
        "index.html"
      );

    if (
      fs.existsSync(
        indexPath
      )
    ) {
      return res.sendFile(
        indexPath
      );
    }

    next();
  }
);

/*
===========================================================
 START
===========================================================
*/

app.listen(
  PORT,
  () => {
    console.log("");
    console.log(
      "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    );

    console.log(
      "✦ CreatorOS"
    );

    console.log(
      "Local AI Creator Operating System"
    );

    console.log(
      "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    );

    console.log(
      `http://localhost:${PORT}`
    );

    console.log(
      `Ollama: ${OLLAMA_URL}`
    );

    console.log(
      `Model: ${MODEL_NAME}`
    );

    console.log(
      `YouTube API: ${
        YT_API_KEY
          ? "configured"
          : "NOT CONFIGURED"
      }`
    );

    console.log(
      `Database: ${dbPath}`
    );

    console.log(
      "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    );

    console.log("");
  }
);

process.on(
  "SIGINT",
  () => {
    try {
      db.close();
    } catch {}

    process.exit(
      0
    );
  }
);

process.on(
  "SIGTERM",
  () => {
    try {
      db.close();
    } catch {}

    process.exit(
      0
    );
  }
);



index.html

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta
  name="viewport"
  content="width=device-width,initial-scale=1.0"
>
<meta
  name="theme-color"
  content="#080811"
>
<title>CreatorOS — Local AI Creator Operating System</title>

<style>
:root{
  --bg:#07070c;
  --bg2:#0b0b13;
  --panel:rgba(18,18,30,.78);
  --panel2:rgba(24,24,40,.82);
  --panel3:rgba(12,12,21,.92);
  --line:rgba(255,255,255,.09);
  --line2:rgba(255,255,255,.14);
  --text:#f5f5fb;
  --muted:#9696a8;
  --muted2:#6e6e82;
  --pink:#ff1764;
  --purple:#7c3aed;
  --violet:#a855f7;
  --cyan:#4de8ff;
  --green:#49efaa;
  --yellow:#ffd166;
  --red:#ff5577;
  --shadow:0 30px 100px rgba(0,0,0,.45);
  --radius:18px;
  --radius2:12px;
  --sidebar:250px;
}

*{
  box-sizing:border-box;
}

html,
body{
  margin:0;
  padding:0;
  min-height:100%;
  background:
    radial-gradient(
      circle at 15% 0%,
      rgba(124,58,237,.15),
      transparent 32%
    ),
    radial-gradient(
      circle at 90% 10%,
      rgba(255,23,100,.10),
      transparent 30%
    ),
    linear-gradient(
      135deg,
      #06060b,
      #090910 45%,
      #07070d
    );
  color:var(--text);
  font-family:
    Inter,
    ui-sans-serif,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    sans-serif;
}

body{
  overflow:hidden;
}

button,
input,
textarea,
select{
  font:inherit;
}

button{
  border:0;
}

a{
  color:inherit;
  text-decoration:none;
}

.app{
  height:100vh;
  display:flex;
  overflow:hidden;
}

/* =====================================================
 SIDEBAR
===================================================== */

.sidebar{
  width:var(--sidebar);
  flex:0 0 var(--sidebar);
  border-right:1px solid var(--line);
  background:
    linear-gradient(
      180deg,
      rgba(13,13,22,.96),
      rgba(7,7,13,.97)
    );
  backdrop-filter:blur(30px);
  display:flex;
  flex-direction:column;
  position:relative;
  z-index:20;
}

.brand{
  padding:22px 20px 18px;
  display:flex;
  gap:12px;
  align-items:center;
  border-bottom:1px solid var(--line);
}

.brand-logo{
  width:38px;
  height:38px;
  border-radius:12px;
  display:grid;
  place-items:center;
  background:
    linear-gradient(
      135deg,
      var(--pink),
      var(--violet)
    );
  box-shadow:
    0 12px 30px rgba(255,23,100,.25);
  font-weight:900;
}

.brand-title{
  font-size:13px;
  font-weight:900;
  letter-spacing:.5px;
}

.brand-sub{
  font-size:8px;
  color:var(--muted);
  margin-top:3px;
  letter-spacing:1px;
  text-transform:uppercase;
}

.nav{
  padding:16px 12px;
  overflow:auto;
}

.nav-section{
  margin:10px 8px 7px;
  font-size:7px;
  color:var(--muted2);
  letter-spacing:1.8px;
  text-transform:uppercase;
  font-weight:800;
}

.nav-btn{
  width:100%;
  padding:11px 12px;
  border-radius:10px;
  color:#a8a8ba;
  background:transparent;
  display:flex;
  align-items:center;
  gap:10px;
  cursor:pointer;
  text-align:left;
  transition:.2s;
  margin-bottom:3px;
  font-size:10px;
}

.nav-btn:hover{
  background:rgba(255,255,255,.045);
  color:#fff;
}

.nav-btn.active{
  background:
    linear-gradient(
      90deg,
      rgba(255,23,100,.15),
      rgba(124,58,237,.12)
    );
  color:#fff;
  box-shadow:
    inset 2px 0 0 var(--pink);
}

.nav-icon{
  width:20px;
  text-align:center;
  font-size:13px;
}

.sidebar-bottom{
  margin-top:auto;
  padding:14px;
  border-top:1px solid var(--line);
}

.local-card{
  padding:12px;
  border-radius:13px;
  background:
    rgba(255,255,255,.025);
  border:1px solid var(--line);
}

.local-row{
  display:flex;
  justify-content:space-between;
  align-items:center;
  gap:8px;
}

.local-title{
  font-size:9px;
  font-weight:800;
}

.local-sub{
  margin-top:4px;
  font-size:7px;
  color:var(--muted);
}

.dot{
  width:7px;
  height:7px;
  border-radius:50%;
  background:var(--green);
  box-shadow:
    0 0 12px var(--green);
}

/* =====================================================
 MAIN
===================================================== */

.main{
  min-width:0;
  flex:1;
  display:flex;
  flex-direction:column;
  overflow:hidden;
}

.topbar{
  min-height:64px;
  padding:10px 18px;
  border-bottom:1px solid var(--line);
  background:
    rgba(7,7,13,.76);
  backdrop-filter:blur(25px);
  display:flex;
  align-items:center;
  gap:12px;
  z-index:10;
}

.mobile-menu{
  display:none;
}

.top-title{
  min-width:0;
  flex:1;
}

.top-title b{
  display:block;
  font-size:12px;
}

.top-title span{
  display:block;
  margin-top:3px;
  font-size:8px;
  color:var(--muted);
}

.status-pill{
  padding:7px 10px;
  border:1px solid var(--line);
  border-radius:999px;
  display:flex;
  align-items:center;
  gap:7px;
  font-size:8px;
  color:#c6c6d4;
  background:rgba(255,255,255,.025);
}

.status-pill .dot{
  width:5px;
  height:5px;
}

.global-mic{
  width:34px;
  height:34px;
  border-radius:10px;
  background:
    rgba(255,255,255,.055);
  color:#fff;
  cursor:pointer;
  border:1px solid var(--line);
  transition:.2s;
}

.global-mic:hover{
  background:
    rgba(255,23,100,.13);
  border-color:
    rgba(255,23,100,.35);
}

.global-mic.recording{
  background:
    rgba(255,23,100,.22);
  box-shadow:
    0 0 0 5px rgba(255,23,100,.08),
    0 0 25px rgba(255,23,100,.25);
  animation:
    pulseMic 1s infinite;
}

@keyframes pulseMic{
  50%{
    transform:scale(1.05);
  }
}

.content{
  flex:1;
  overflow:auto;
  padding:20px;
}

.view{
  display:none;
  animation:
    fadeIn .22s ease;
}

.view.active{
  display:block;
}

@keyframes fadeIn{
  from{
    opacity:0;
    transform:translateY(5px);
  }

  to{
    opacity:1;
    transform:none;
  }
}

/* =====================================================
 COMMON
===================================================== */

.panel{
  background:
    linear-gradient(
      145deg,
      rgba(20,20,34,.86),
      rgba(10,10,18,.92)
    );
  border:1px solid var(--line);
  border-radius:var(--radius);
  box-shadow:var(--shadow);
  backdrop-filter:blur(25px);
}

.section-head{
  display:flex;
  align-items:flex-end;
  justify-content:space-between;
  gap:16px;
  margin-bottom:14px;
}

.section-head h2{
  margin:0;
  font-size:18px;
  letter-spacing:-.5px;
}

.section-head p{
  margin:5px 0 0;
  font-size:9px;
  color:var(--muted);
}

.grid{
  display:grid;
  gap:14px;
}

.grid-2{
  grid-template-columns:
    minmax(0,1.35fr)
    minmax(300px,.65fr);
}

.grid-3{
  grid-template-columns:
    repeat(3,minmax(0,1fr));
}

.btn{
  padding:9px 13px;
  border-radius:10px;
  cursor:pointer;
  color:#fff;
  font-size:9px;
  font-weight:800;
  border:1px solid var(--line);
  transition:.2s;
}

.btn:hover{
  transform:translateY(-1px);
}

.btn.primary{
  background:
    linear-gradient(
      135deg,
      var(--pink),
      var(--purple)
    );
  box-shadow:
    0 10px 25px rgba(124,58,237,.20);
}

.btn.secondary{
  background:
    rgba(255,255,255,.045);
}

.btn.ghost{
  background:transparent;
}

.btn.danger{
  background:
    rgba(255,55,90,.10);
  border-color:
    rgba(255,55,90,.25);
  color:#ff8297;
}

.input,
.textarea,
.select{
  width:100%;
  color:#f7f7fc;
  background:
    rgba(255,255,255,.035);
  border:1px solid var(--line);
  border-radius:11px;
  outline:none;
  transition:.2s;
}

.input{
  height:40px;
  padding:0 12px;
}

.textarea{
  min-height:120px;
  padding:12px;
  resize:vertical;
  line-height:1.55;
  font-size:10px;
}

.select{
  height:40px;
  padding:0 10px;
}

.input:focus,
.textarea:focus,
.select:focus{
  border-color:
    rgba(168,85,247,.55);
  box-shadow:
    0 0 0 3px rgba(168,85,247,.08);
}

label{
  display:block;
  margin:0 0 6px;
  color:#c9c9d5;
  font-size:8px;
  font-weight:800;
  text-transform:uppercase;
  letter-spacing:1px;
}

.empty{
  min-height:150px;
  display:grid;
  place-items:center;
  text-align:center;
  color:var(--muted);
  font-size:9px;
  padding:25px;
}

.tag{
  display:inline-flex;
  align-items:center;
  padding:5px 7px;
  border-radius:999px;
  font-size:7px;
  border:1px solid var(--line);
  color:#c4c4d2;
  background:rgba(255,255,255,.025);
}

.muted{
  color:var(--muted);
}

/* =====================================================
 COMMAND CENTER
===================================================== */

.hero{
  min-height:230px;
  padding:26px;
  position:relative;
  overflow:hidden;
  background:
    radial-gradient(
      circle at 75% 20%,
      rgba(255,23,100,.18),
      transparent 35%
    ),
    radial-gradient(
      circle at 30% 100%,
      rgba(124,58,237,.20),
      transparent 38%
    ),
    linear-gradient(
      135deg,
      rgba(24,20,39,.94),
      rgba(10,10,19,.96)
    );
}

.hero::after{
  content:"";
  position:absolute;
  width:280px;
  height:280px;
  right:-100px;
  bottom:-150px;
  border-radius:50%;
  border:1px solid rgba(255,255,255,.08);
  box-shadow:
    0 0 100px rgba(168,85,247,.18);
}

.hero-eyebrow{
  font-size:7px;
  text-transform:uppercase;
  letter-spacing:2px;
  color:#d5b9ff;
  font-weight:900;
}

.hero h1{
  margin:9px 0 0;
  max-width:760px;
  font-size:31px;
  line-height:1.05;
  letter-spacing:-1.5px;
}

.hero h1 span{
  background:
    linear-gradient(
      90deg,
      #fff,
      #c9a8ff,
      #ff79a6
    );
  -webkit-background-clip:text;
  color:transparent;
}

.hero p{
  max-width:680px;
  color:#aaaabd;
  font-size:10px;
  line-height:1.6;
}

.command-row{
  display:flex;
  gap:8px;
  margin-top:18px;
  max-width:850px;
}

.command-row .input{
  flex:1;
}

.mic-button{
  width:42px;
  flex:0 0 42px;
  border-radius:10px;
  background:
    rgba(255,255,255,.045);
  color:#fff;
  border:1px solid var(--line);
  cursor:pointer;
}

.mic-button.recording{
  background:
    rgba(255,23,100,.18);
  border-color:
    rgba(255,23,100,.35);
  animation:
    pulseMic 1s infinite;
}

.quick-chips{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  margin-top:10px;
}

.quick-chip{
  border:1px solid var(--line);
  background:
    rgba(255,255,255,.025);
  color:#aaaabc;
  border-radius:999px;
  padding:6px 8px;
  cursor:pointer;
  font-size:7px;
}

.quick-chip:hover{
  color:#fff;
  border-color:
    rgba(168,85,247,.4);
}

.metric-grid{
  display:grid;
  grid-template-columns:
    repeat(4,minmax(0,1fr));
  gap:10px;
  margin-top:14px;
}

.metric{
  padding:13px;
}

.metric small{
  display:block;
  font-size:7px;
  color:var(--muted);
  text-transform:uppercase;
  letter-spacing:1px;
}

.metric strong{
  display:block;
  font-size:17px;
  margin-top:7px;
}

.ai-panel{
  margin-top:14px;
  padding:16px;
}

.ai-head{
  display:flex;
  align-items:center;
  justify-content:space-between;
  gap:10px;
}

.ai-head b{
  font-size:10px;
}

.ai-status{
  color:var(--green);
  font-size:7px;
  letter-spacing:1px;
}

.ai-output{
  margin-top:12px;
  min-height:170px;
  max-height:480px;
  overflow:auto;
  white-space:pre-wrap;
  font-family:
    ui-monospace,
    SFMono-Regular,
    Menlo,
    Consolas,
    monospace;
  color:#ddddea;
  font-size:9px;
  line-height:1.7;
  padding:13px;
  background:
    rgba(0,0,0,.18);
  border:1px solid var(--line);
  border-radius:12px;
}

.cursor{
  border-right:
    1px solid var(--cyan);
  animation:
    blink .8s infinite;
}

@keyframes blink{
  50%{
    border-color:transparent;
  }
}

/* =====================================================
 QUESTION YOUTUBE
===================================================== */

.related-panel{
  margin-top:14px;
  padding:16px;
}

.related-head{
  display:flex;
  justify-content:space-between;
  align-items:center;
  gap:12px;
}

.related-head h3{
  margin:0;
  font-size:11px;
}

.related-head p{
  margin:4px 0 0;
  font-size:7px;
  color:var(--muted);
}

.video-strip{
  display:grid;
  grid-template-columns:
    repeat(
      auto-fit,
      minmax(190px,1fr)
    );
  gap:10px;
  margin-top:12px;
}

.video-card{
  overflow:hidden;
  border:1px solid var(--line);
  border-radius:13px;
  background:
    rgba(255,255,255,.025);
  transition:.2s;
}

.video-card:hover{
  transform:translateY(-2px);
  border-color:
    rgba(168,85,247,.4);
}

.video-thumb{
  position:relative;
  aspect-ratio:16/9;
  background:#000;
  overflow:hidden;
}

.video-thumb img{
  width:100%;
  height:100%;
  object-fit:cover;
  display:block;
}

.video-duration{
  position:absolute;
  right:6px;
  bottom:6px;
  background:rgba(0,0,0,.82);
  color:#fff;
  padding:3px 5px;
  border-radius:4px;
  font-size:7px;
}

.video-body{
  padding:9px;
}

.video-title{
  font-size:8px;
  font-weight:800;
  line-height:1.35;
  min-height:32px;
}

.video-channel{
  margin-top:4px;
  color:var(--muted);
  font-size:7px;
}

.video-actions{
  display:flex;
  gap:5px;
  margin-top:8px;
}

.video-actions .btn{
  padding:6px 7px;
  font-size:7px;
}

/* =====================================================
 CREATOR STUDIO
===================================================== */

.format-grid{
  display:grid;
  grid-template-columns:
    repeat(
      4,
      minmax(0,1fr)
    );
  gap:9px;
  margin-bottom:14px;
}

.format-card{
  padding:13px;
  border-radius:13px;
  background:
    rgba(255,255,255,.025);
  border:1px solid var(--line);
  cursor:pointer;
  transition:.2s;
}

.format-card:hover{
  border-color:
    rgba(168,85,247,.4);
}

.format-card.active{
  background:
    linear-gradient(
      135deg,
      rgba(255,23,100,.12),
      rgba(124,58,237,.13)
    );
  border-color:
    rgba(255,23,100,.38);
  box-shadow:
    0 10px 30px rgba(124,58,237,.10);
}

.format-icon{
  font-size:18px;
}

.format-card strong{
  display:block;
  margin-top:7px;
  font-size:9px;
}

.format-card span{
  display:block;
  margin-top:4px;
  font-size:7px;
  color:var(--muted);
  line-height:1.4;
}

.studio-layout{
  display:grid;
  grid-template-columns:
    minmax(0,1.4fr)
    minmax(280px,.6fr);
  gap:14px;
}

.form-panel{
  padding:16px;
}

.form-grid{
  display:grid;
  grid-template-columns:
    repeat(2,minmax(0,1fr));
  gap:12px;
}

.full{
  grid-column:1/-1;
}

.form-actions{
  display:flex;
  gap:7px;
  flex-wrap:wrap;
  margin-top:12px;
}

.creator-output{
  margin-top:14px;
  min-height:300px;
  max-height:680px;
  overflow:auto;
  white-space:pre-wrap;
  padding:15px;
  background:
    rgba(0,0,0,.18);
  border:1px solid var(--line);
  border-radius:13px;
  color:#e4e4ef;
  font-size:9px;
  line-height:1.65;
}

.side-stack{
  display:flex;
  flex-direction:column;
  gap:12px;
}

.side-card{
  padding:15px;
}

.side-card h3{
  margin:0;
  font-size:10px;
}

.side-card p{
  color:var(--muted);
  font-size:8px;
  line-height:1.55;
}

.pipeline{
  margin-top:10px;
}

.pipeline-step{
  display:flex;
  gap:8px;
  padding:7px 0;
  border-bottom:1px solid var(--line);
  font-size:8px;
}

.pipeline-step:last-child{
  border-bottom:0;
}

.pipeline-dot{
  width:7px;
  height:7px;
  border-radius:50%;
  margin-top:3px;
  background:
    var(--purple);
  box-shadow:
    0 0 10px rgba(124,58,237,.7);
}

/* =====================================================
 YOUTUBE LAB
===================================================== */

.youtube-search{
  display:flex;
  gap:7px;
  margin-bottom:13px;
}

.youtube-search .input{
  flex:1;
}

.search-mic{
  width:42px;
  flex:0 0 42px;
  border-radius:10px;
  border:1px solid var(--line);
  background:
    rgba(255,255,255,.045);
  color:#fff;
  cursor:pointer;
}

.search-mic.recording{
  background:
    rgba(255,23,100,.20);
  border-color:
    rgba(255,23,100,.45);
  animation:
    pulseMic 1s infinite;
}

.youtube-layout{
  display:grid;
  grid-template-columns:
    minmax(0,1.6fr)
    minmax(300px,.8fr);
  gap:14px;
}

.player-panel{
  overflow:hidden;
}

.player-box{
  width:100%;
  aspect-ratio:16/9;
  min-height:300px;
  background:#000;
  position:relative;
  overflow:hidden;
}

.player-box iframe,
.player-box > div{
  position:absolute !important;
  inset:0 !important;
  width:100% !important;
  height:100% !important;
  border:0 !important;
}

.player-info{
  padding:13px;
  border-top:1px solid var(--line);
}

.player-info h3{
  margin:0;
  font-size:11px;
}

.player-info p{
  margin:4px 0 0;
  color:var(--muted);
  font-size:8px;
}

.player-controls{
  padding:9px 12px;
  display:flex;
  gap:5px;
  flex-wrap:wrap;
  border-top:1px solid var(--line);
}

.player-controls .btn{
  padding:6px 8px;
  font-size:7px;
}

.youtube-side{
  display:flex;
  flex-direction:column;
  gap:12px;
  min-height:0;
}

.results-panel{
  padding:13px;
  min-height:0;
  max-height:510px;
  overflow:auto;
}

.results-head{
  display:flex;
  justify-content:space-between;
  align-items:center;
  margin-bottom:9px;
}

.results-head b{
  font-size:9px;
}

.result-item{
  display:flex;
  gap:8px;
  padding:8px 0;
  border-bottom:1px solid var(--line);
}

.result-item:last-child{
  border-bottom:0;
}

.result-thumb{
  width:95px;
  flex:0 0 95px;
  aspect-ratio:16/9;
  border-radius:7px;
  overflow:hidden;
  background:#000;
}

.result-thumb img{
  width:100%;
  height:100%;
  object-fit:cover;
}

.result-info{
  min-width:0;
}

.result-info strong{
  display:block;
  font-size:7px;
  line-height:1.35;
}

.result-info span{
  display:block;
  margin-top:3px;
  color:var(--muted);
  font-size:6px;
}

.result-buttons{
  display:flex;
  gap:4px;
  margin-top:5px;
}

.result-buttons button{
  font-size:6px;
  padding:4px 6px;
  border-radius:6px;
  background:
    rgba(255,255,255,.05);
  color:#ddd;
  border:1px solid var(--line);
  cursor:pointer;
}

/* =====================================================
 RESEARCH
===================================================== */

.research-panel{
  overflow:hidden;
  min-height:300px;
}

.research-tabs{
  display:flex;
  gap:0;
  overflow:auto;
  border-bottom:1px solid var(--line);
}

.research-tab{
  padding:10px 12px;
  background:transparent;
  color:var(--muted);
  font-size:7px;
  cursor:pointer;
  border-bottom:2px solid transparent;
  white-space:nowrap;
}

.research-tab.active{
  color:#fff;
  border-bottom-color:
    var(--pink);
}

.research-body{
  padding:13px;
  max-height:500px;
  overflow:auto;
}

.transcript{
  white-space:pre-wrap;
  color:#d8d8e6;
  font-size:8px;
  line-height:1.7;
}

.research-actions{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  margin-bottom:10px;
}

.metadata-grid{
  display:grid;
  grid-template-columns:
    repeat(2,minmax(0,1fr));
  gap:8px;
}

.meta{
  padding:9px;
  border:1px solid var(--line);
  border-radius:9px;
  background:
    rgba(255,255,255,.025);
}

.meta small{
  display:block;
  color:var(--muted2);
  font-size:6px;
  text-transform:uppercase;
  letter-spacing:1px;
}

.meta b{
  display:block;
  margin-top:4px;
  font-size:8px;
  line-height:1.4;
}

.social-list{
  display:grid;
  gap:7px;
}

.social-link{
  display:block;
  padding:10px;
  border:1px solid var(--line);
  border-radius:10px;
  background:
    rgba(255,255,255,.025);
}

.social-link:hover{
  border-color:
    rgba(168,85,247,.4);
}

.social-link b{
  display:block;
  font-size:8px;
}

.social-link span{
  display:block;
  margin-top:3px;
  color:var(--muted);
  font-size:7px;
}

/* =====================================================
 VIDEO INTELLIGENCE
===================================================== */

.intel-tabs{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  margin-bottom:12px;
}

.intel-tab{
  padding:7px 9px;
  border-radius:8px;
  background:
    rgba(255,255,255,.035);
  color:var(--muted);
  border:1px solid var(--line);
  cursor:pointer;
  font-size:7px;
}

.intel-tab.active{
  background:
    rgba(124,58,237,.16);
  color:#fff;
  border-color:
    rgba(124,58,237,.35);
}

.intel-output{
  min-height:400px;
  max-height:650px;
  overflow:auto;
  padding:15px;
  white-space:pre-wrap;
  color:#ddddea;
  font-size:9px;
  line-height:1.7;
  background:
    rgba(0,0,0,.16);
  border:1px solid var(--line);
  border-radius:12px;
}

/* =====================================================
 MEMORY / ACTIVITY
===================================================== */

.list{
  display:grid;
  gap:8px;
}

.list-item{
  padding:11px;
  border:1px solid var(--line);
  border-radius:10px;
  background:
    rgba(255,255,255,.025);
}

.list-item b{
  font-size:8px;
}

.list-item span{
  display:block;
  margin-top:4px;
  color:var(--muted);
  font-size:7px;
  line-height:1.45;
}

.history-card{
  padding:14px;
}

.history-card h3{
  margin:0;
  font-size:10px;
}

.history-card p{
  margin:5px 0 0;
  color:var(--muted);
  font-size:7px;
}

.history-content{
  margin-top:10px;
  white-space:pre-wrap;
  max-height:300px;
  overflow:auto;
  padding:10px;
  border:1px solid var(--line);
  border-radius:9px;
  background:rgba(0,0,0,.16);
  font-size:8px;
  line-height:1.5;
}

/* =====================================================
 TOAST
===================================================== */

.toast{
  position:fixed;
  right:18px;
  bottom:18px;
  z-index:100;
  min-width:220px;
  max-width:360px;
  padding:11px 13px;
  border-radius:11px;
  background:
    rgba(18,18,28,.96);
  border:1px solid var(--line2);
  box-shadow:
    0 20px 50px rgba(0,0,0,.45);
  color:#eee;
  font-size:8px;
  transform:
    translateY(15px);
  opacity:0;
  pointer-events:none;
  transition:.25s;
}

.toast.show{
  transform:none;
  opacity:1;
}

/* =====================================================
 MOBILE
===================================================== */

@media(max-width:1050px){
  .grid-2,
  .studio-layout,
  .youtube-layout{
    grid-template-columns:1fr;
  }

  .format-grid{
    grid-template-columns:
      repeat(2,minmax(0,1fr));
  }
}

@media(max-width:760px){
  body{
    overflow:auto;
  }

  .app{
    min-height:100vh;
    height:auto;
  }

  .sidebar{
    position:fixed;
    inset:0 auto 0 0;
    transform:
      translateX(-105%);
    transition:.25s;
    box-shadow:
      20px 0 60px rgba(0,0,0,.5);
  }

  .sidebar.open{
    transform:none;
  }

  .mobile-menu{
    display:block;
    width:34px;
    height:34px;
    border-radius:9px;
    border:1px solid var(--line);
    background:
      rgba(255,255,255,.04);
    color:#fff;
  }

  .content{
    padding:13px;
  }

  .metric-grid{
    grid-template-columns:
      repeat(2,minmax(0,1fr));
  }

  .hero h1{
    font-size:25px;
  }

  .command-row{
    flex-wrap:wrap;
  }

  .youtube-search{
    flex-wrap:wrap;
  }

  .youtube-search .input{
    min-width:0;
    width:100%;
    flex-basis:100%;
  }

  .youtube-layout{
    display:flex;
    flex-direction:column;
  }

  .player-box{
    min-height:0;
  }

  .form-grid{
    grid-template-columns:1fr;
  }

  .full{
    grid-column:auto;
  }
}

@media(max-width:500px){
  .topbar{
    padding:8px 10px;
  }

  .status-pill{
    display:none;
  }

  .format-grid{
    grid-template-columns:1fr;
  }

  .metadata-grid{
    grid-template-columns:1fr;
  }

  .hero{
    padding:18px;
  }
}
</style>
</head>

<body>

<div class="app">

  <!-- =================================================
       SIDEBAR
  ================================================== -->

  <aside
    class="sidebar"
    id="sidebar"
  >

    <div class="brand">
      <div class="brand-logo">
        ✦
      </div>

      <div>
        <div class="brand-title">
          CreatorOS
        </div>

        <div class="brand-sub">
          Local AI Creator System
        </div>
      </div>
    </div>

    <nav class="nav">

      <div class="nav-section">
        Workspace
      </div>

      <button
        class="nav-btn active"
        data-view="home"
      >
        <span class="nav-icon">⌂</span>
        Command Center
      </button>

      <button
        class="nav-btn"
        data-view="studio"
      >
        <span class="nav-icon">✦</span>
        Creator Studio
      </button>

      <button
        class="nav-btn"
        data-view="youtube"
      >
        <span class="nav-icon">▶</span>
        YouTube Lab
      </button>

      <button
        class="nav-btn"
        data-view="intelligence"
      >
        <span class="nav-icon">◈</span>
        Video Intelligence
      </button>

      <div class="nav-section">
        Knowledge
      </div>

      <button
        class="nav-btn"
        data-view="memory"
      >
        <span class="nav-icon">⌘</span>
        Memory & Signals
      </button>

      <button
        class="nav-btn"
        data-view="history"
      >
        <span class="nav-icon">▤</span>
        Project History
      </button>

    </nav>

    <div class="sidebar-bottom">

      <div class="local-card">

        <div class="local-row">
          <div class="local-title">
            Local AI
          </div>

          <div
            class="dot"
            id="sidebarDot"
          ></div>
        </div>

        <div
          class="local-sub"
          id="sidebarModel"
        >
          Checking Ollama...
        </div>

      </div>

    </div>

  </aside>

  <!-- =================================================
       MAIN
  ================================================== -->

  <main class="main">

    <header class="topbar">

      <button
        class="mobile-menu"
        id="mobileMenu"
      >
        ☰
      </button>

      <div class="top-title">
        <b id="topTitle">
          Command Center
        </b>

        <span id="topSubtitle">
          Ask anything. Create anything.
        </span>
      </div>

      <div
        class="status-pill"
        id="ollamaStatus"
      >
        <span class="dot"></span>
        <span>
          Ollama
        </span>
      </div>

      <div
        class="status-pill"
        id="modelStatus"
      >
        <span>
          phi3:latest
        </span>
      </div>

      <button
        class="global-mic"
        id="globalMic"
        title="Voice input"
      >
        🎙
      </button>

    </header>

    <div class="content">

      <!-- =================================================
           COMMAND CENTER
      ================================================== -->

      <section
        class="view active"
        id="view-home"
      >

        <div class="panel hero">

          <div class="hero-eyebrow">
            LOCAL AI · CREATOR COMMAND CENTER
          </div>

          <h1>
            Think.
            <span>
              Research.
            </span>
            Create.
          </h1>

          <p>
            Ask CreatorOS a question and Ollama
            streams the answer locally. Related
            YouTube research appears alongside the
            answer so you can understand, watch and
            research without leaving the workspace.
          </p>

          <div class="command-row">

            <input
              class="input"
              id="missionInput"
              placeholder="Ask CreatorOS anything..."
              autocomplete="off"
            >

            <button
              class="mic-button"
              id="missionMic"
              title="Speak your question"
            >
              🎙
            </button>

            <button
              class="btn primary"
              id="askBtn"
            >
              Ask CreatorOS
            </button>

          </div>

          <div class="quick-chips">

            <button
              class="quick-chip"
              data-question="Give me 10 YouTube Shorts ideas about AI tools"
            >
              AI Shorts ideas
            </button>

            <button
              class="quick-chip"
              data-question="Explain how to create a viral YouTube Short"
            >
              Viral Short workflow
            </button>

            <button
              class="quick-chip"
              data-question="Give me a complete long-form YouTube video structure about financial crime"
            >
              Long video
            </button>

            <button
              class="quick-chip"
              data-question="Give me emotional Gujarati devotional Shorts ideas"
            >
              Devotional ideas
            </button>

          </div>

        </div>

        <div class="metric-grid">

          <div class="panel metric">
            <small>
              AI ENGINE
            </small>

            <strong id="metricModel">
              phi3
            </strong>
          </div>

          <div class="panel metric">
            <small>
              YOUTUBE
            </small>

            <strong id="metricYoutube">
              Ready
            </strong>
          </div>

          <div class="panel metric">
            <small>
              WORKSPACE
            </small>

            <strong>
              Local
            </strong>
          </div>

          <div class="panel metric">
            <small>
              STREAMING
            </small>

            <strong>
              SSE
            </strong>
          </div>

        </div>

        <div class="panel ai-panel">

          <div class="ai-head">

            <b>
              Live Ollama Answer
            </b>

            <span
              class="ai-status"
              id="aiStatus"
            >
              READY
            </span>

          </div>

          <div
            class="ai-output"
            id="aiOutput"
          >
            Ask a question above.

            CreatorOS will stream the answer from your
            local Ollama model.
          </div>

        </div>

        <div
          class="panel related-panel"
          id="questionYoutubePanel"
        >

          <div class="related-head">

            <div>
              <h3>
                YouTube Research
              </h3>

              <p id="questionYoutubeSub">
                Ask a question to discover related videos.
              </p>
            </div>

            <button
              class="btn secondary"
              id="openYoutubeLab"
            >
              Open YouTube Lab
            </button>

          </div>

          <div
            class="video-strip"
            id="questionVideos"
          >
            <div class="empty">
              Related YouTube videos will appear here
              after you ask CreatorOS a question.
            </div>
          </div>

        </div>

      </section>

      <!-- =================================================
           CREATOR STUDIO
      ================================================== -->

      <section
        class="view"
        id="view-studio"
      >

        <div class="section-head">

          <div>
            <h2>
              Creator Studio
            </h2>

            <p>
              Generate format-aware content using local Ollama.
            </p>
          </div>

          <span class="tag">
            LOCAL GENERATION
          </span>

        </div>

        <div
          class="format-grid"
          id="formatGrid"
        ></div>

        <div class="studio-layout">

          <div class="panel form-panel">

            <div class="form-grid">

              <div class="full">

                <label>
                  Topic / Creative Brief
                </label>

                <textarea
                  class="textarea"
                  id="creatorTopic"
                  placeholder="What do you want to create?"
                ></textarea>

              </div>

              <div>

                <label>
                  Language
                </label>

                <select
                  class="select"
                  id="creatorLanguage"
                >
                  <option>
                    English
                  </option>

                  <option>
                    Hindi
                  </option>

                  <option>
                    Gujarati
                  </option>

                  <option>
                    Hinglish
                  </option>
                </select>

              </div>

              <div>

                <label>
                  Style
                </label>

                <select
                  class="select"
                  id="creatorStyle"
                >
                  <option>
                    Modern & engaging
                  </option>

                  <option>
                    Emotional storytelling
                  </option>

                  <option>
                    Cinematic
                  </option>

                  <option>
                    Educational
                  </option>

                  <option>
                    Devotional & respectful
                  </option>

                  <option>
                    Fast-paced viral
                  </option>

                  <option>
                    Professional / LinkedIn
                  </option>
                </select>

              </div>

              <div>

                <label>
                  Duration
                </label>

                <input
                  class="input"
                  id="creatorDuration"
                  placeholder="e.g. 60 sec"
                >

              </div>

              <div>

                <label>
                  Number of Shorts
                </label>

                <input
                  class="input"
                  id="shortsCount"
                  type="number"
                  min="3"
                  max="10"
                  value="5"
                >

              </div>

              <div class="full">

                <label>
                  Source / Long Script / Transcript
                </label>

                <textarea
                  class="textarea"
                  id="creatorSource"
                  placeholder="Paste a long script or transcript here for Long → Shorts..."
                ></textarea>

              </div>

            </div>

            <div class="form-actions">

              <button
                class="btn primary"
                id="generateCreator"
              >
                ✦ Generate with Local AI
              </button>

              <button
                class="btn secondary"
                id="saveCreator"
              >
                Save Project
              </button>

              <button
                class="btn secondary"
                id="copyCreator"
              >
                Copy
              </button>

            </div>

            <div
              class="creator-output"
              id="creatorOutput"
            >
              Your generated creator content will appear here.
            </div>

          </div>

          <div class="side-stack">

            <div class="panel side-card">

              <h3>
                Production Pipeline
              </h3>

              <p>
                CreatorOS keeps the production process
                structured instead of putting everything
                into one giant page.
              </p>

              <div class="pipeline">

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    Brief
                  </span>
                </div>

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    Format selection
                  </span>
                </div>

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    Local AI generation
                  </span>
                </div>

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    Repurposing
                  </span>
                </div>

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    YouTube research
                  </span>
                </div>

                <div class="pipeline-step">
                  <span class="pipeline-dot"></span>
                  <span>
                    Save to project
                  </span>
                </div>

              </div>

            </div>

            <div class="panel side-card">

              <h3>
                Smart Repurposing
              </h3>

              <p>
                Choose
                <b>
                  Long → Shorts
                </b>
                and paste your long script or transcript.
                CreatorOS will create multiple independent
                Shorts instead of simply cutting the text.
              </p>

            </div>

            <div class="panel side-card">

              <h3>
                Local AI
              </h3>

              <p>
                Reasoning and generation use Ollama.
                No OpenAI API key is required.
              </p>

            </div>

          </div>

        </div>

      </section>

      <!-- =================================================
           YOUTUBE LAB
      ================================================== -->

      <section
        class="view"
        id="view-youtube"
      >

        <div class="section-head">

          <div>
            <h2>
              YouTube Lab
            </h2>

            <p>
              Search, voice-search, play, research and analyze
              YouTube videos without leaving CreatorOS.
            </p>
          </div>

          <span class="tag">
            VIDEO RESEARCH
          </span>

        </div>

        <div class="youtube-search">

          <input
            class="input"
            id="youtubeSearch"
            placeholder="Search YouTube..."
            autocomplete="off"
          >

          <button
            class="search-mic"
            id="youtubeMic"
            title="Voice search YouTube"
          >
            🎙
          </button>

          <button
            class="btn primary"
            id="youtubeSearchBtn"
          >
            Search
          </button>

        </div>

        <div class="youtube-layout">

          <div>

            <div class="panel player-panel">

              <div class="player-box">

                <div
                  id="youtubePlayer"
                ></div>

              </div>

              <div class="player-controls">

                <button
                  class="btn secondary"
                  data-player="back30"
                >
                  ↶ 30s
                </button>

                <button
                  class="btn secondary"
                  data-player="back10"
                >
                  ↶ 10s
                </button>

                <button
                  class="btn primary"
                  data-player="play"
                >
                  ▶ Play
                </button>

                <button
                  class="btn secondary"
                  data-player="pause"
                >
                  Pause
                </button>

                <button
                  class="btn secondary"
                  data-player="forward10"
                >
                  10s ↷
                </button>

                <button
                  class="btn secondary"
                  data-player="forward30"
                >
                  30s ↷
                </button>

                <button
                  class="btn secondary"
                  data-player="restart"
                >
                  Restart
                </button>

                <button
                  class="btn secondary"
                  data-player="fullscreen"
                >
                  Fullscreen
                </button>

              </div>

              <div class="player-info">

                <div
                  style="
                    display:flex;
                    justify-content:space-between;
                    gap:10px;
                    align-items:flex-start
                  "
                >

                  <div
                    style="
                      min-width:0
                    "
                  >

                    <h3
                      id="selectedTitle"
                    >
                      No video selected
                    </h3>

                    <p
                      id="selectedChannel"
                    >
                      Search and select a YouTube video.
                    </p>

                    <div
                      id="transcriptStatus"
                      style="
                        margin-top:6px;
                        color:var(--muted);
                        font-size:7px
                      "
                    >
                      Transcript not loaded.
                    </div>

                  </div>

                  <button
                    class="btn secondary"
                    id="openSelectedYoutube"
                  >
                    ↗ YouTube
                  </button>

                </div>

              </div>

            </div>

            <div
              class="panel research-panel"
              style="
                margin-top:14px
              "
            >

              <div
                class="research-tabs"
              >

                <button
                  class="research-tab active"
                  data-research="transcript"
                >
                  Transcript
                </button>

                <button
                  class="research-tab"
                  data-research="metadata"
                >
                  Metadata
                </button>

                <button
                  class="research-tab"
                  data-research="social"
                >
                  Social / X
                </button>

              </div>

              <div
                class="research-body"
                id="researchBody"
              >

                <div class="empty">
                  Select a video to load its
                  transcript and metadata.
                </div>

              </div>

            </div>

          </div>

          <div class="youtube-side">

            <div
              class="panel results-panel"
            >

              <div class="results-head">

                <b>
                  Search Results
                </b>

                <span
                  class="tag"
                  id="searchResultCount"
                >
                  0
                </span>

              </div>

              <div
                id="youtubeResults"
              >
                <div class="empty">
                  Search YouTube to see results.
                </div>
              </div>

            </div>

            <div
              class="panel results-panel"
            >

              <div class="results-head">

                <b>
                  Current Playlist
                </b>

                <button
                  class="btn secondary"
                  id="clearPlaylist"
                >
                  Clear
                </button>

              </div>

              <div
                id="playlistList"
              >
                <div class="empty">
                  Playlist is empty.
                </div>
              </div>

            </div>

          </div>

        </div>

      </section>

      <!-- =================================================
           VIDEO INTELLIGENCE
      ================================================== -->

      <section
        class="view"
        id="view-intelligence"
      >

        <div class="section-head">

          <div>
            <h2>
              Video Intelligence
            </h2>

            <p>
              Use the selected transcript as grounded input
              for local AI analysis.
            </p>
          </div>

          <button
            class="btn secondary"
            id="goYoutubeFromIntel"
          >
            Open YouTube Lab
          </button>

        </div>

        <div
          class="panel"
          style="
            padding:14px;
            margin-bottom:12px
          "
        >

          <div
            class="intel-tabs"
            id="intelTabs"
          >

            <button
              class="intel-tab active"
              data-intel="summary"
            >
              Summary
            </button>

            <button
              class="intel-tab"
              data-intel="analysis"
            >
              Deep Analysis
            </button>

            <button
              class="intel-tab"
              data-intel="notes"
            >
              Research Notes
            </button>

            <button
              class="intel-tab"
              data-intel="quiz"
            >
              Quiz
            </button>

            <button
              class="intel-tab"
              data-intel="shorts"
            >
              Shorts
            </button>

            <button
              class="intel-tab"
              data-intel="title"
            >
              Titles
            </button>

            <button
              class="intel-tab"
              data-intel="hashtags"
            >
              Hashtags
            </button>

            <button
              class="intel-tab"
              data-intel="linkedin"
            >
              LinkedIn
            </button>

          </div>

          <button
            class="btn primary"
            id="runIntel"
          >
            Run Local AI Analysis
          </button>

        </div>

        <div class="panel">

          <div
            class="intel-output"
            id="intelOutput"
          >
            Select a YouTube video, load its transcript,
            then run one of the local AI analysis actions.
          </div>

        </div>

      </section>

      <!-- =================================================
           MEMORY
      ================================================== -->

      <section
        class="view"
        id="view-memory"
      >

        <div class="section-head">

          <div>
            <h2>
              Memory & Signals
            </h2>

            <p>
              Local creator memory, activity and saved signals.
            </p>
          </div>

        </div>

        <div class="grid grid-2">

          <div class="panel form-panel">

            <label>
              Save Memory
            </label>

            <textarea
              class="textarea"
              id="memoryInput"
              placeholder="Example: My audience responds strongly to emotional Gujarati Shorts."
            ></textarea>

            <button
              class="btn primary"
              id="saveMemory"
              style="
                margin-top:9px
              "
            >
              Save Memory
            </button>

            <div
              style="
                margin-top:15px
              "
            >

              <label>
                Saved Memory
              </label>

              <div
                class="list"
                id="memoryList"
              ></div>

            </div>

          </div>

          <div class="panel form-panel">

            <label>
              Activity
            </label>

            <div
              class="list"
              id="activityList"
            ></div>

          </div>

        </div>

      </section>

      <!-- =================================================
           HISTORY
      ================================================== -->

      <section
        class="view"
        id="view-history"
      >

        <div class="section-head">

          <div>
            <h2>
              Project History
            </h2>

            <p>
              Saved Creator Studio projects.
            </p>
          </div>

          <button
            class="btn secondary"
            id="refreshProjects"
          >
            Refresh
          </button>

        </div>

        <div
          class="grid"
          id="projectList"
          style="
            grid-template-columns:
              repeat(
                auto-fit,
                minmax(280px,1fr)
              )
          "
        >

          <div class="empty">
            No saved projects yet.
          </div>

        </div>

      </section>

    </div>

  </main>

</div>

<div
  class="toast"
  id="toast"
></div>

<script src="https://www.youtube.com/iframe_api"></script>

<script>
"use strict";

/* =====================================================
 GLOBAL STATE
===================================================== */

const state = {
  view: "home",

  selected: null,

  transcript: "",

  metadata: null,

  player: null,

  playerReady: false,

  results: [],

  playlist: [],

  selectedResultIndex: -1,

  selectedFormat: "short",

  researchTab: "transcript",

  intelligenceAction: "summary",

  lastQuestion: "",

  lastAI: "",

  recognition: null,

  recordingTarget: null,

  projects: []
};

/* =====================================================
 HELPERS
===================================================== */

const $ = selector =>
  document.querySelector(
    selector
  );

const $$ = selector =>
  Array.from(
    document.querySelectorAll(
      selector
    )
  );

function escapeHTML(
  value
) {
  return String(
    value ?? ""
  )
    .replace(
      /&/g,
      "&amp;"
    )
    .replace(
      /</g,
      "&lt;"
    )
    .replace(
      />/g,
      "&gt;"
    )
    .replace(
      /"/g,
      "&quot;"
    )
    .replace(
      /'/g,
      "&#039;"
    );
}

function fmtNum(
  value
) {
  const n =
    Number(value || 0);

  if (
    n >= 10000000
  ) {
    return (
      (n / 10000000)
        .toFixed(1)
        .replace(
          /\.0$/,
          ""
        ) +
      "Cr"
    );
  }

  if (
    n >= 100000
  ) {
    return (
      (n / 100000)
        .toFixed(1)
        .replace(
          /\.0$/,
          ""
        ) +
      "L"
    );
  }

  if (
    n >= 1000
  ) {
    return (
      (n / 1000)
        .toFixed(1)
        .replace(
          /\.0$/,
          ""
        ) +
      "K"
    );
  }

  return String(
    n
  );
}

function normalizeVideo(
  video = {}
) {
  return {
    id:
      video.id ||
      video.videoId ||
      "",

    title:
      video.title ||
      "Untitled video",

    channel:
      video.channel ||
      video.channelTitle ||
      "",

    thumbnail:
      video.thumbnail ||
      `https://i.ytimg.com/vi/${encodeURIComponent(
        video.id ||
          video.videoId ||
          ""
      )}/hqdefault.jpg`,

    duration:
      video.duration ||
      "",

    views:
      Number(
        video.views || 0
      ),

    likes:
      Number(
        video.likes || 0
      ),

    publishedAt:
      video.publishedAt ||
      ""
  };
}

function toast(
  message
) {
  const el =
    $("#toast");

  if (!el) return;

  el.textContent =
    message;

  el.classList.add(
    "show"
  );

  clearTimeout(
    toast.timer
  );

  toast.timer =
    setTimeout(
      () => {
        el.classList.remove(
          "show"
        );
      },
      2800
    );
}

async function api(
  url,
  options = {}
) {
  const response =
    await fetch(
      url,
      {
        ...options,

        headers: {
          "Content-Type":
            "application/json",

          ...(options.headers ||
            {})
        }
      }
    );

  let data;

  try {
    data =
      await response.json();
  } catch {
    data = {};
  }

  if (
    !response.ok
  ) {
    throw new Error(
      data.error ||
      `HTTP ${response.status}`
    );
  }

  return data;
}

/* =====================================================
 NAVIGATION
===================================================== */

const viewTitles = {
  home: [
    "Command Center",
    "Ask anything. Create anything."
  ],

  studio: [
    "Creator Studio",
    "Format-aware local AI content generation."
  ],

  youtube: [
    "YouTube Lab",
    "Search, watch and research without leaving CreatorOS."
  ],

  intelligence: [
    "Video Intelligence",
    "Transcript-grounded local AI analysis."
  ],

  memory: [
    "Memory & Signals",
    "Local creator memory and activity."
  ],

  history: [
    "Project History",
    "Your saved Creator Studio work."
  ]
};

function showView(
  view
) {
  state.view =
    view;

  $$(".view")
    .forEach(
      el => {
        el.classList.toggle(
          "active",
          el.id ===
            `view-${view}`
        );
      }
    );

  $$(".nav-btn")
    .forEach(
      btn => {
        btn.classList.toggle(
          "active",
          btn.dataset.view ===
            view
        );
      }
    );

  const info =
    viewTitles[view] ||
    viewTitles.home;

  $("#topTitle")
    .textContent =
    info[0];

  $("#topSubtitle")
    .textContent =
    info[1];

  $("#sidebar")
    .classList.remove(
      "open"
    );

  if (
    view ===
    "youtube"
  ) {
    loadPlaylist();
  }

  if (
    view ===
    "memory"
  ) {
    loadMemory();
    loadActivity();
  }

  if (
    view ===
    "history"
  ) {
    loadProjects();
  }
}

$$(".nav-btn")
  .forEach(
    btn => {
      btn.addEventListener(
        "click",
        () => {
          showView(
            btn.dataset.view
          );
        }
      );
    }
  );

$("#mobileMenu")
  .addEventListener(
    "click",
    () => {
      $("#sidebar")
        .classList.toggle(
          "open"
        );
    }
  );

/* =====================================================
 HEALTH
===================================================== */

async function loadHealth() {
  try {
    const data =
      await api(
        "/api/health"
      );

    const online =
      data.ollama?.online;

    $("#ollamaStatus")
      .innerHTML =
      `
        <span
          class="dot"
          style="
            background:${
              online
                ? "var(--green)"
                : "var(--red)"
            };
            box-shadow:0 0 12px ${
              online
                ? "var(--green)"
                : "var(--red)"
            }
          "
        ></span>

        <span>
          ${
            online
              ? "Ollama Online"
              : "Ollama Offline"
          }
        </span>
      `;

    $("#modelStatus")
      .textContent =
      data.model ||
      "phi3:latest";

    $("#metricModel")
      .textContent =
      data.model ||
      "phi3";

    $("#metricYoutube")
      .textContent =
      data.youtube
        ? "Ready"
        : "API key needed";

    $("#sidebarModel")
      .textContent =
      `${data.model || "phi3:latest"} · ${
        online
          ? "online"
          : "offline"
      }`;

    $("#sidebarDot")
      .style.background =
      online
        ? "var(--green)"
        : "var(--red)";

  } catch (
    error
  ) {
    $("#ollamaStatus")
      .innerHTML =
      `
        <span
          class="dot"
          style="
            background:var(--red);
            box-shadow:0 0 12px var(--red)
          "
        ></span>

        <span>
          Offline
        </span>
      `;

    $("#sidebarModel")
      .textContent =
      "Ollama unavailable";
  }
}

/* =====================================================
 STREAM SSE
===================================================== */

async function streamEndpoint(
  url,
  body,
  onChunk,
  onComplete,
  onError
) {
  const response =
    await fetch(
      url,
      {
        method:
          "POST",

        headers: {
          "Content-Type":
            "application/json"
        },

        body:
          JSON.stringify(body)
      }
    );

  if (
    !response.ok
  ) {
    let message =
      `HTTP ${response.status}`;

    try {
      const data =
        await response.json();

      message =
        data.error ||
        message;
    } catch {}

    throw new Error(
      message
    );
  }

  if (
    !response.body
  ) {
    throw new Error(
      "Streaming response unavailable."
    );
  }

  const reader =
    response.body.getReader();

  const decoder =
    new TextDecoder();

  let buffer =
    "";

  let completed =
    false;

  while (
    true
  ) {
    const {
      value,
      done
    } =
      await reader.read();

    if (
      done
    ) {
      break;
    }

    buffer +=
      decoder.decode(
        value,
        {
          stream:
            true
        }
      );

    const blocks =
      buffer.split(
        "\n\n"
      );

    buffer =
      blocks.pop() ||
      "";

    for (
      const block of blocks
    ) {
      let event =
        "message";

      let dataText =
        "";

      const lines =
        block.split(
          "\n"
        );

      for (
        const line of lines
      ) {
        if (
          line.startsWith(
            "event:"
          )
        ) {
          event =
            line
              .slice(6)
              .trim();
        }

        if (
          line.startsWith(
            "data:"
          )
        ) {
          dataText +=
            line
              .slice(5)
              .trim();
        }
      }

      if (
        !dataText
      ) {
        continue;
      }

      let data;

      try {
        data =
          JSON.parse(
            dataText
          );
      } catch {
        continue;
      }

      if (
        event ===
        "ai_chunk"
      ) {
        onChunk(
          data.text ||
            ""
        );
      }

      if (
        event ===
        "ai_error"
      ) {
        if (
          onError
        ) {
          onError(
            data.error ||
              "AI error"
          );
        }
      }

      if (
        event ===
        "ai_complete"
      ) {
        completed =
          true;
      }
    }
  }

  if (
    completed ||
    !buffer
  ) {
    if (
      onComplete
    ) {
      onComplete();
    }
  }
}

/* =====================================================
 MAIN QUESTION
===================================================== */

async function askCreator() {
  const input =
    $("#missionInput")
      .value
      .trim();

  if (
    !input
  ) {
    toast(
      "Ask a question first."
    );

    return;
  }

  state.lastQuestion =
    input;

  $("#aiOutput")
    .textContent =
    "";

  $("#aiStatus")
    .textContent =
    "THINKING · OLLAMA";

  $("#aiOutput")
    .classList.add(
      "cursor"
    );

  try {
    await streamEndpoint(
      "/api/creator/stream",

      {
        question:
          input
      },

      text => {
        $("#aiOutput")
          .textContent +=
          text;

        const output =
          $("#aiOutput");

        output.scrollTop =
          output.scrollHeight;
      },

      () => {
        $("#aiStatus")
          .textContent =
          "COMPLETE · LOCAL AI";

        $("#aiOutput")
          .classList.remove(
            "cursor"
          );

        state.lastAI =
          $("#aiOutput")
            .textContent;

        /*
        Automatically search YouTube
        using the same user question.
        */

        loadQuestionYouTube(
          input
        );
      },

      error => {
        $("#aiStatus")
          .textContent =
          "AI ERROR";

        $("#aiOutput")
          .textContent =
          error;

        $("#aiOutput")
          .classList.remove(
            "cursor"
          );
      }
    );
  } catch (
    error
  ) {
    $("#aiStatus")
      .textContent =
      "ERROR";

    $("#aiOutput")
      .textContent =
      error.message;

    $("#aiOutput")
      .classList.remove(
        "cursor"
      );
  }
}

$("#askBtn")
  .addEventListener(
    "click",
    askCreator
  );

$("#missionInput")
  .addEventListener(
    "keydown",
    event => {
      if (
        event.key ===
          "Enter" &&
        (
          event.ctrlKey ||
          event.metaKey
        )
      ) {
        askCreator();
      }
    }
  );

$$(".quick-chip")
  .forEach(
    chip => {
      chip.addEventListener(
        "click",
        () => {
          $("#missionInput")
            .value =
            chip.dataset.question;

          askCreator();
        }
      );
    }
  );

/* =====================================================
 QUESTION → YOUTUBE
===================================================== */

async function loadQuestionYouTube(
  question
) {
  const container =
    $("#questionVideos");

  $("#questionYoutubeSub")
    .textContent =
    "Finding videos related to your question...";

  container.innerHTML =
    `
      <div class="empty">
        Searching YouTube...
      </div>
    `;

  try {
    const data =
      await api(
        "/api/youtube/recommend",
        {
          method:
            "POST",

          body:
            JSON.stringify({
              question
            })
        }
      );

    $("#questionYoutubeSub")
      .textContent =
      `Search: ${
        data.query ||
        question
      }`;

    renderVideoStrip(
      container,
      data.videos ||
        []
    );
  } catch (
    error
  ) {
    container.innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;

    $("#questionYoutubeSub")
      .textContent =
      "YouTube recommendations unavailable.";
  }
}

function renderVideoStrip(
  container,
  videos
) {
  if (
    !videos.length
  ) {
    container.innerHTML =
      `
        <div class="empty">
          No related YouTube videos found.
        </div>
      `;

    return;
  }

  container.innerHTML =
    videos
      .map(
        video => {
          const v =
            normalizeVideo(
              video
            );

          return `
            <article
              class="video-card"
            >

              <div
                class="video-thumb"
              >
                <img
                  src="${escapeHTML(
                    v.thumbnail
                  )}"
                  alt=""
                  loading="lazy"
                >

                ${
                  v.duration
                    ? `
                      <span
                        class="video-duration"
                      >
                        ${escapeHTML(
                          v.duration
                        )}
                      </span>
                    `
                    : ""
                }
              </div>

              <div
                class="video-body"
              >

                <div
                  class="video-title"
                >
                  ${escapeHTML(
                    v.title
                  )}
                </div>

                <div
                  class="video-channel"
                >
                  ${escapeHTML(
                    v.channel
                  )}
                  ·
                  ${fmtNum(
                    v.views
                  )} views
                </div>

                <div
                  class="video-actions"
                >

                  <button
                    class="btn primary"
                    data-play-video="${escapeHTML(
                      v.id
                    )}"
                  >
                    ▶ Play Here
                  </button>

                  <button
                    class="btn secondary"
                    data-research-video="${escapeHTML(
                      v.id
                    )}"
                  >
                    Research
                  </button>

                  <button
                    class="btn secondary"
                    data-open-video="${escapeHTML(
                      v.id
                    )}"
                  >
                    ↗
                  </button>

                </div>

              </div>

            </article>
          `;
        }
      )
      .join("");

  container
    .querySelectorAll(
      "[data-play-video]"
    )
    .forEach(
      button => {
        button.addEventListener(
          "click",
          () => {
            const video =
              videos.find(
                item =>
                  String(
                    item.id
                  ) ===
                  String(
                    button.dataset
                      .playVideo
                  )
              );

            if (
              video
            ) {
              selectVideo(
                video,
                true
              );
            }
          }
        );
      }
    );

  container
    .querySelectorAll(
      "[data-research-video]"
    )
    .forEach(
      button => {
        button.addEventListener(
          "click",
          () => {
            const video =
              videos.find(
                item =>
                  String(
                    item.id
                  ) ===
                  String(
                    button.dataset
                      .researchVideo
                  )
              );

            if (
              video
            ) {
              selectVideo(
                video,
                true
              );

              showView(
                "intelligence"
              );
            }
          }
        );
      }
    );

  container
    .querySelectorAll(
      "[data-open-video]"
    )
    .forEach(
      button => {
        button.addEventListener(
          "click",
          () => {
            window.open(
              `https://www.youtube.com/watch?v=${encodeURIComponent(
                button.dataset
                  .openVideo
              )}`,
              "_blank",
              "noopener"
            );
          }
        );
      }
    );
}

/* =====================================================
 CREATOR FORMATS
===================================================== */

async function loadCreatorFormats() {
  try {
    const data =
      await api(
        "/api/creator/formats"
      );

    const formats =
      data.formats ||
      [];

    $("#formatGrid")
      .innerHTML =
      formats
        .map(
          format => `
            <div
              class="format-card ${
                format.id ===
                state.selectedFormat
                  ? "active"
                  : ""
              }"
              data-format="${escapeHTML(
                format.id
              )}"
            >

              <div
                class="format-icon"
              >
                ${escapeHTML(
                  format.icon
                )}
              </div>

              <strong>
                ${escapeHTML(
                  format.label
                )}
              </strong>

              <span>
                ${escapeHTML(
                  format.description
                )}
              </span>

            </div>
          `
        )
        .join("");

    $$(".format-card")
      .forEach(
        card => {
          card.addEventListener(
            "click",
            () => {
              state.selectedFormat =
                card.dataset
                  .format;

              $$(".format-card")
                .forEach(
                  item =>
                    item.classList.toggle(
                      "active",
                      item ===
                        card
                    )
                );
            }
          );
        }
      );
  } catch (
    error
  ) {
    $("#formatGrid")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

/* =====================================================
 CREATOR GENERATION
===================================================== */

async function generateCreator() {
  const topic =
    $("#creatorTopic")
      .value
      .trim();

  const sourceText =
    $("#creatorSource")
      .value
      .trim();

  if (
    !topic &&
    !sourceText
  ) {
    toast(
      "Add a topic or source transcript."
    );

    return;
  }

  $("#creatorOutput")
    .textContent =
    "";

  try {
    await streamEndpoint(
      "/api/creator/generate-stream",

      {
        topic,

        sourceText,

        format:
          state.selectedFormat,

        language:
          $("#creatorLanguage")
            .value,

        style:
          $("#creatorStyle")
            .value,

        duration:
          $("#creatorDuration")
            .value
            .trim(),

        shortsCount:
          Number(
            $("#shortsCount")
              .value ||
              5
          )
      },

      text => {
        $("#creatorOutput")
          .textContent +=
          text;

        $("#creatorOutput")
          .scrollTop =
          $("#creatorOutput")
            .scrollHeight;
      },

      () => {
        toast(
          "Creator content generated."
        );
      },

      error => {
        $("#creatorOutput")
          .textContent =
          error;
      }
    );
  } catch (
    error
  ) {
    $("#creatorOutput")
      .textContent =
      error.message;
  }
}

$("#generateCreator")
  .addEventListener(
    "click",
    generateCreator
  );

$("#copyCreator")
  .addEventListener(
    "click",
    async () => {
      const text =
        $("#creatorOutput")
          .textContent;

      try {
        await navigator.clipboard.writeText(
          text
        );

        toast(
          "Copied."
        );
      } catch {
        toast(
          "Copy failed."
        );
      }
    }
  );

$("#saveCreator")
  .addEventListener(
    "click",
    saveCreatorProject
  );

async function saveCreatorProject() {
  const content =
    $("#creatorOutput")
      .textContent
      .trim();

  if (
    !content
  ) {
    toast(
      "Generate something first."
    );

    return;
  }

  try {
    await api(
      "/api/creator/projects",
      {
        method:
          "POST",

        body:
          JSON.stringify({
            title:
              $("#creatorTopic")
                .value
                .trim() ||
              "CreatorOS Project",

            topic:
              $("#creatorTopic")
                .value
                .trim(),

            format:
              state.selectedFormat,

            content
          })
      }
    );

    toast(
      "Project saved."
    );
  } catch (
    error
  ) {
    toast(
      error.message
    );
  }
}

/* =====================================================
 YOUTUBE PLAYER
===================================================== */

function onYouTubeIframeAPIReady() {
  createPlayer();
}

window.onYouTubeIframeAPIReady =
  onYouTubeIframeAPIReady;

function createPlayer() {
  if (
    state.player ||
    !window.YT ||
    !YT.Player
  ) {
    return;
  }

  state.player =
    new YT.Player(
      "youtubePlayer",
      {
        width:
          "100%",

        height:
          "100%",

        videoId:
          "",

        playerVars: {
          autoplay:
            0,

          controls:
            1,

          rel:
            0,

          modestbranding:
            1,

          playsinline:
            1,

          enablejsapi:
            1
        },

        events: {
          onReady:
            () => {
              state.playerReady =
                true;

              if (
                state.selected
              ) {
                state.player
                  .cueVideoById(
                    state.selected
                      .id
                  );
              }
            },

          onError:
            event => {
              toast(
                `YouTube player error: ${event.data}`
              );
            },

          onStateChange:
            event => {
              if (
                window.YT &&
                event.data ===
                  YT.PlayerState.ENDED
              ) {
                const index =
                  state.results.findIndex(
                    video =>
                      video.id ===
                      state.selected
                        ?.id
                  );

                if (
                  index >=
                    0 &&
                  index <
                    state.results.length -
                      1
                ) {
                  selectVideo(
                    state.results[
                      index +
                        1
                    ],
                    false
                  );
                }
              }
            }
        }
      }
    );
}

/* =====================================================
 SELECT VIDEO
===================================================== */

async function selectVideo(
  video,
  openLab = true
) {
  const v =
    normalizeVideo(
      video
    );

  if (
    !v.id
  ) {
    return;
  }

  state.selected =
    v;

  state.transcript =
    "";

  state.metadata =
    null;

  $("#selectedTitle")
    .textContent =
    v.title;

  $("#selectedChannel")
    .textContent =
    `${v.channel || "Unknown channel"} · ${fmtNum(
      v.views
    )} views`;

  $("#transcriptStatus")
    .textContent =
    "Loading transcript + metadata...";

  if (
    openLab
  ) {
    showView(
      "youtube"
    );
  }

  renderResearch();

  if (
    !state.player &&
    window.YT &&
    YT.Player
  ) {
    createPlayer();
  }

  if (
    state.playerReady &&
    state.player
  ) {
    state.player
      .cueVideoById(
        v.id
      );
  }

  await loadVideoContext(
    v.id
  );
}

async function loadVideoContext(
  id
) {
  try {
    const data =
      await api(
        `/api/youtube/context/${encodeURIComponent(
          id
        )}`
      );

    state.metadata =
      data.video ||
      null;

    state.transcript =
      data.transcript ||
      "";

    if (
      data.transcriptAvailable
    ) {
      $("#transcriptStatus")
        .textContent =
        `Transcript ready · ${state.transcript.length.toLocaleString()} characters`;
    } else {
      $("#transcriptStatus")
        .textContent =
        `Transcript unavailable · ${
          data.transcriptError ||
          "No transcript returned."
        }`;
    }

    renderResearch();
  } catch (
    error
  ) {
    $("#transcriptStatus")
      .textContent =
      `Research error · ${error.message}`;

    renderResearch();
  }
}

/* =====================================================
 YOUTUBE SEARCH
===================================================== */

async function searchYouTubeUI(
  query
) {
  const q =
    (
      query ||
      $("#youtubeSearch")
        .value
    )
      .trim();

  if (
    !q
  ) {
    toast(
      "Enter a YouTube search."
    );

    return;
  }

  $("#youtubeResults")
    .innerHTML =
    `
      <div class="empty">
        Searching YouTube...
      </div>
    `;

  try {
    const data =
      await api(
        `/api/youtube/search?q=${encodeURIComponent(
          q
        )}&limit=12`
      );

    state.results =
      (
        data.videos ||
        []
      ).map(
        normalizeVideo
      );

    $("#searchResultCount")
      .textContent =
      state.results.length;

    renderSearchResults();
  } catch (
    error
  ) {
    $("#youtubeResults")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

$("#youtubeSearchBtn")
  .addEventListener(
    "click",
    () =>
      searchYouTubeUI()
  );

$("#youtubeSearch")
  .addEventListener(
    "keydown",
    event => {
      if (
        event.key ===
        "Enter"
      ) {
        searchYouTubeUI();
      }
    }
  );

function renderSearchResults() {
  const container =
    $("#youtubeResults");

  if (
    !state.results.length
  ) {
    container.innerHTML =
      `
        <div class="empty">
          No videos found.
        </div>
      `;

    return;
  }

  container.innerHTML =
    state.results
      .map(
        (video, index) => `
          <div
            class="result-item"
          >

            <div
              class="result-thumb"
            >
              <img
                src="${escapeHTML(
                  video.thumbnail
                )}"
                alt=""
                loading="lazy"
              >
            </div>

            <div
              class="result-info"
            >

              <strong>
                ${escapeHTML(
                  video.title
                )}
              </strong>

              <span>
                ${escapeHTML(
                  video.channel
                )}
                ·
                ${fmtNum(
                  video.views
                )} views
              </span>

              <div
                class="result-buttons"
              >

                <button
                  data-result-play="${index}"
                >
                  ▶ Play
                </button>

                <button
                  data-result-research="${index}"
                >
                  Research
                </button>

                <button
                  data-result-open="${index}"
                >
                  ↗
                </button>

                <button
                  data-result-add="${index}"
                >
                  + Playlist
                </button>

              </div>

            </div>

          </div>
        `
      )
      .join("");

  $$(
    "[data-result-play]"
  ).forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          const video =
            state.results[
              Number(
                button.dataset
                  .resultPlay
              )
            ];

          selectVideo(
            video,
            false
          );
        }
      );
    }
  );

  $$(
    "[data-result-research]"
  ).forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          const video =
            state.results[
              Number(
                button.dataset
                  .resultResearch
              )
            ];

          selectVideo(
            video,
            true
          );
        }
      );
    }
  );

  $$(
    "[data-result-open]"
  ).forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          const video =
            state.results[
              Number(
                button.dataset
                  .resultOpen
              )
            ];

          window.open(
            `https://www.youtube.com/watch?v=${encodeURIComponent(
              video.id
            )}`,
            "_blank",
            "noopener"
          );
        }
      );
    }
  );

  $$(
    "[data-result-add]"
  ).forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          const video =
            state.results[
              Number(
                button.dataset
                  .resultAdd
              )
            ];

          addToPlaylist(
            video
          );
        }
      );
    }
  );
}

/* =====================================================
 RESEARCH TABS
===================================================== */

$$(".research-tab")
  .forEach(
    tab => {
      tab.addEventListener(
        "click",
        () => {
          state.researchTab =
            tab.dataset
              .research;

          $$(".research-tab")
            .forEach(
              item =>
                item.classList.toggle(
                  "active",
                  item ===
                    tab
                )
            );

          renderResearch();
        }
      );
    }
  );

function renderResearch() {
  const body =
    $("#researchBody");

  if (
    !body
  ) {
    return;
  }

  if (
    !state.selected
  ) {
    body.innerHTML =
      `
        <div class="empty">
          Select a YouTube video first.
        </div>
      `;

    return;
  }

  if (
    state.researchTab ===
    "transcript"
  ) {
    body.innerHTML =
      `
        <div
          class="research-actions"
        >

          <button
            class="btn secondary"
            id="copyTranscript"
          >
            Copy Transcript
          </button>

          <button
            class="btn secondary"
            id="openTranscriptYoutube"
          >
            ↗ Open YouTube
          </button>

        </div>

        ${
          state.transcript
            ? `
              <div
                class="transcript"
              >
                ${escapeHTML(
                  state.transcript
                )}
              </div>
            `
            : `
              <div class="empty">
                ${
                  state.metadata
                    ? "No transcript is available for this video."
                    : "Loading transcript..."
                }
              </div>
            `
        }
      `;

    $("#copyTranscript")
      ?.addEventListener(
        "click",
        async () => {
          if (
            !state.transcript
          ) {
            toast(
              "No transcript available."
            );

            return;
          }

          try {
            await navigator.clipboard.writeText(
              state.transcript
            );

            toast(
              "Transcript copied."
            );
          } catch {
            toast(
              "Copy failed."
            );
          }
        }
      );

    $("#openTranscriptYoutube")
      ?.addEventListener(
        "click",
        () => {
          window.open(
            `https://www.youtube.com/watch?v=${encodeURIComponent(
              state.selected.id
            )}`,
            "_blank",
            "noopener"
          );
        }
      );

    return;
  }

  if (
    state.researchTab ===
    "metadata"
  ) {
    const m =
      state.metadata ||
      state.selected;

    body.innerHTML =
      `
        <div
          class="metadata-grid"
        >

          <div class="meta">
            <small>
              Title
            </small>

            <b>
              ${escapeHTML(
                m.title ||
                  "—"
              )}
            </b>
          </div>

          <div class="meta">
            <small>
              Channel
            </small>

            <b>
              ${escapeHTML(
                m.channel ||
                  "—"
              )}
            </b>
          </div>

          <div class="meta">
            <small>
              Duration
            </small>

            <b>
              ${escapeHTML(
                m.duration ||
                  "—"
              )}
            </b>
          </div>

          <div class="meta">
            <small>
              Views
            </small>

            <b>
              ${fmtNum(
                m.views
              )}
            </b>
          </div>

          <div class="meta">
            <small>
              Likes
            </small>

            <b>
              ${fmtNum(
                m.likes
              )}
            </b>
          </div>

          <div class="meta">
            <small>
              Published
            </small>

            <b>
              ${
                m.publishedAt
                  ? new Date(
                      m.publishedAt
                    ).toLocaleDateString()
                  : "—"
              }
            </b>
          </div>

        </div>

        <div
          style="
            margin-top:10px
          "
        >

          <div
            class="meta"
          >

            <small>
              Description
            </small>

            <div
              style="
                margin-top:7px;
                color:#d0d0df;
                font-size:8px;
                line-height:1.6;
                white-space:pre-wrap
              "
            >
              ${escapeHTML(
                m.description ||
                  "No description available."
              )}
            </div>

          </div>

        </div>

        <div
          style="
            margin-top:8px
          "
        >

          <div
            class="meta"
          >

            <small>
              Tags
            </small>

            <div
              style="
                margin-top:7px;
                color:#d0d0df;
                font-size:8px;
                line-height:1.5
              "
            >
              ${
                (
                  m.tags ||
                  []
                )
                  .map(
                    tag =>
                      `#${escapeHTML(
                        tag
                      )}`
                  )
                  .join(
                    " · "
                  ) ||
                "No tags available."
              }
            </div>

          </div>

        </div>
      `;

    return;
  }

  const title =
    state.metadata?.title ||
    state.selected?.title ||
    "YouTube video";

  const channel =
    state.metadata?.channel ||
    state.selected?.channel ||
    "";

  const query =
    encodeURIComponent(
      `${title} ${channel}`
    );

  const videoId =
    encodeURIComponent(
      state.selected.id
    );

  body.innerHTML =
    `
      <div
        class="social-list"
      >

        <a
          class="social-link"
          href="https://www.youtube.com/watch?v=${videoId}"
          target="_blank"
          rel="noopener noreferrer"
        >
          <b>
            ▶ YouTube
          </b>

          <span>
            Open the original video.
          </span>
        </a>

        <a
          class="social-link"
          href="https://x.com/search?q=${query}&src=typed_query"
          target="_blank"
          rel="noopener noreferrer"
        >
          <b>
            𝕏 / Twitter Search
          </b>

          <span>
            Search public posts related to this video.
          </span>
        </a>

        <a
          class="social-link"
          href="https://www.youtube.com/results?search_query=${query}"
          target="_blank"
          rel="noopener noreferrer"
        >
          <b>
            Related YouTube Search
          </b>

          <span>
            Open broader YouTube results.
          </span>
        </a>

        <a
          class="social-link"
          href="https://www.google.com/search?q=${query}+YouTube"
          target="_blank"
          rel="noopener noreferrer"
        >
          <b>
            Web Research
          </b>

          <span>
            Search the video title across the web.
          </span>
        </a>

      </div>

      <div
        style="
          margin-top:10px;
          padding:10px;
          border:1px solid var(--line);
          border-radius:10px;
          color:var(--muted);
          font-size:7px;
          line-height:1.6
        "
      >
        CreatorOS provides search links here.
        It does not claim that individual social posts
        were retrieved or verified.
      </div>
    `;
}

/* =====================================================
 OPEN SELECTED YOUTUBE
===================================================== */

$("#openSelectedYoutube")
  .addEventListener(
    "click",
    () => {
      if (
        !state.selected
      ) {
        toast(
          "Select a video first."
        );

        return;
      }

      window.open(
        `https://www.youtube.com/watch?v=${encodeURIComponent(
          state.selected.id
        )}`,
        "_blank",
        "noopener"
      );
    }
  );

/* =====================================================
 PLAYER CONTROLS
===================================================== */

$$("[data-player]")
  .forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          if (
            !state.player
          ) {
            toast(
              "YouTube player is not ready."
            );

            return;
          }

          const action =
            button.dataset
              .player;

          try {
            if (
              action ===
              "play"
            ) {
              state.player
                .playVideo();
            }

            if (
              action ===
              "pause"
            ) {
              state.player
                .pauseVideo();
            }

            if (
              action ===
              "restart"
            ) {
              state.player
                .seekTo(
                  0,
                  true
                );

              state.player
                .playVideo();
            }

            if (
              action ===
              "back10"
            ) {
              seekBy(
                -10
              );
            }

            if (
              action ===
              "back30"
            ) {
              seekBy(
                -30
              );
            }

            if (
              action ===
              "forward10"
            ) {
              seekBy(
                10
              );
            }

            if (
              action ===
              "forward30"
            ) {
              seekBy(
                30
              );
            }

            if (
              action ===
              "fullscreen"
            ) {
              const frame =
                document.querySelector(
                  "#youtubePlayer iframe"
                );

              if (
                frame?.requestFullscreen
              ) {
                frame.requestFullscreen();
              }
            }
          } catch {}
        }
      );
    }
  );

function seekBy(
  seconds
) {
  if (
    !state.player
  ) {
    return;
  }

  const current =
    Number(
      state.player
        .getCurrentTime() ||
        0
    );

  state.player
    .seekTo(
      Math.max(
        0,
        current +
          seconds
      ),
      true
    );
}

/* =====================================================
 PLAYLIST
===================================================== */

async function loadPlaylist() {
  try {
    const data =
      await api(
        "/api/playlist"
      );

    state.playlist =
      (
        data.playlist ||
        []
      ).map(
        normalizeVideo
      );

    renderPlaylist();
  } catch (
    error
  ) {
    $("#playlistList")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

function renderPlaylist() {
  const container =
    $("#playlistList");

  if (
    !state.playlist.length
  ) {
    container.innerHTML =
      `
        <div class="empty">
          Playlist is empty.
        </div>
      `;

    return;
  }

  container.innerHTML =
    state.playlist
      .map(
        (video, index) => `
          <div
            class="result-item"
          >

            <div
              class="result-thumb"
            >
              <img
                src="${escapeHTML(
                  video.thumbnail
                )}"
                alt=""
              >
            </div>

            <div
              class="result-info"
            >

              <strong>
                ${escapeHTML(
                  video.title
                )}
              </strong>

              <span>
                ${escapeHTML(
                  video.channel
                )}
              </span>

              <div
                class="result-buttons"
              >

                <button
                  data-playlist-play="${index}"
                >
                  ▶
                </button>

              </div>

            </div>

          </div>
        `
      )
      .join("");

  $$(
    "[data-playlist-play]"
  ).forEach(
    button => {
      button.addEventListener(
        "click",
        () => {
          const video =
            state.playlist[
              Number(
                button.dataset
                  .playlistPlay
              )
            ];

          selectVideo(
            video,
            false
          );
        }
      );
    }
  );
}

async function addToPlaylist(
  video
) {
  try {
    await api(
      "/api/playlist",
      {
        method:
          "POST",

        body:
          JSON.stringify({
            video
          })
      }
    );

    toast(
      "Added to playlist."
    );

    await loadPlaylist();
  } catch (
    error
  ) {
    toast(
      error.message
    );
  }
}

$("#clearPlaylist")
  .addEventListener(
    "click",
    async () => {
      try {
        await api(
          "/api/playlist/clear",
          {
            method:
              "POST"
          }
        );

        state.playlist =
          [];

        renderPlaylist();

        toast(
          "Playlist cleared."
        );
      } catch (
        error
      ) {
        toast(
          error.message
        );
      }
    }
  );

/* =====================================================
 VIDEO INTELLIGENCE
===================================================== */

$$(".intel-tab")
  .forEach(
    tab => {
      tab.addEventListener(
        "click",
        () => {
          state.intelligenceAction =
            tab.dataset
              .intel;

          $$(".intel-tab")
            .forEach(
              item =>
                item.classList.toggle(
                  "active",
                  item ===
                    tab
                )
            );
        }
      );
    }
  );

$("#runIntel")
  .addEventListener(
    "click",
    runIntelligence
  );

async function runIntelligence() {
  if (
    !state.selected
  ) {
    toast(
      "Select a YouTube video first."
    );

    showView(
      "youtube"
    );

    return;
  }

  if (
    !state.transcript
  ) {
    toast(
      "Transcript is required."
    );

    showView(
      "youtube"
    );

    return;
  }

  $("#intelOutput")
    .textContent =
    "";

  try {
    await streamEndpoint(
      "/api/youtube/ai",

      {
        action:
          state.intelligenceAction,

        video:
          state.metadata ||
          state.selected,

        transcript:
          state.transcript
      },

      text => {
        $("#intelOutput")
          .textContent +=
          text;

        $("#intelOutput")
          .scrollTop =
          $("#intelOutput")
            .scrollHeight;
      },

      () => {
        toast(
          "Video intelligence complete."
        );
      },

      error => {
        $("#intelOutput")
          .textContent =
          error;
      }
    );
  } catch (
    error
  ) {
    $("#intelOutput")
      .textContent =
      error.message;
  }
}

$("#goYoutubeFromIntel")
  .addEventListener(
    "click",
    () =>
      showView(
        "youtube"
      )
  );

/* =====================================================
 MEMORY
===================================================== */

async function loadMemory() {
  try {
    const data =
      await api(
        "/api/memory"
      );

    const memory =
      data.memory ||
      [];

    $("#memoryList")
      .innerHTML =
      memory.length
        ? memory
            .map(
              item => `
                <div
                  class="list-item"
                >
                  <b>
                    Memory
                  </b>

                  <span>
                    ${escapeHTML(
                      item.text
                    )}
                  </span>

                  <span>
                    ${new Date(
                      item.createdAt
                    ).toLocaleString()}
                  </span>
                </div>
              `
            )
            .join("")
        : `
            <div class="empty">
              No saved memory.
            </div>
          `;
  } catch (
    error
  ) {
    $("#memoryList")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

$("#saveMemory")
  .addEventListener(
    "click",
    async () => {
      const text =
        $("#memoryInput")
          .value
          .trim();

      if (
        !text
      ) {
        toast(
          "Enter memory first."
        );

        return;
      }

      try {
        await api(
          "/api/memory",
          {
            method:
              "POST",

            body:
              JSON.stringify({
                text
              })
          }
        );

        $("#memoryInput")
          .value =
          "";

        toast(
          "Memory saved."
        );

        loadMemory();
      } catch (
        error
      ) {
        toast(
          error.message
        );
      }
    }
  );

/* =====================================================
 ACTIVITY
===================================================== */

async function loadActivity() {
  try {
    const data =
      await api(
        "/api/activity"
      );

    const activity =
      data.activity ||
      [];

    $("#activityList")
      .innerHTML =
      activity.length
        ? activity
            .slice(
              0,
              40
            )
            .map(
              item => `
                <div
                  class="list-item"
                >

                  <b>
                    ${escapeHTML(
                      item.type
                    )}
                  </b>

                  <span>
                    ${escapeHTML(
                      JSON.stringify(
                        item.data ||
                          {}
                      )
                    )}
                  </span>

                  <span>
                    ${new Date(
                      item.createdAt
                    ).toLocaleString()}
                  </span>

                </div>
              `
            )
            .join("")
        : `
            <div class="empty">
              No activity yet.
            </div>
          `;
  } catch (
    error
  ) {
    $("#activityList")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

/* =====================================================
 PROJECT HISTORY
===================================================== */

async function loadProjects() {
  try {
    const data =
      await api(
        "/api/creator/projects"
      );

    state.projects =
      data.projects ||
      [];

    const container =
      $("#projectList");

    if (
      !state.projects.length
    ) {
      container.innerHTML =
        `
          <div class="empty">
            No saved projects yet.
          </div>
        `;

      return;
    }

    container.innerHTML =
      state.projects
        .map(
          project => `
            <div
              class="panel history-card"
            >

              <h3>
                ${escapeHTML(
                  project.title
                )}
              </h3>

              <p>
                ${escapeHTML(
                  project.format
                )}
                ·
                ${new Date(
                  project.createdAt
                ).toLocaleString()}
              </p>

              <p>
                ${escapeHTML(
                  project.topic ||
                    ""
                )}
              </p>

              <div
                class="history-content"
              >
                ${escapeHTML(
                  project.content
                )}
              </div>

              <div
                style="
                  display:flex;
                  gap:6px;
                  margin-top:9px
                "
              >

                <button
                  class="btn secondary"
                  data-open-project="${project.id}"
                >
                  Open in Studio
                </button>

              </div>

            </div>
          `
        )
        .join("");

    $$(
      "[data-open-project]"
    ).forEach(
      button => {
        button.addEventListener(
          "click",
          () => {
            const project =
              state.projects.find(
                item =>
                  String(
                    item.id
                  ) ===
                  String(
                    button.dataset
                      .openProject
                  )
              );

            if (
              !project
            ) {
              return;
            }

            $("#creatorTopic")
              .value =
              project.topic ||
              "";

            state.selectedFormat =
              project.format ||
              "short";

            $("#creatorOutput")
              .textContent =
              project.content ||
              "";

            showView(
              "studio"
            );

            loadCreatorFormats();
          }
        );
      }
    );
  } catch (
    error
  ) {
    $("#projectList")
      .innerHTML =
      `
        <div class="empty">
          ${escapeHTML(
            error.message
          )}
        </div>
      `;
  }
}

$("#refreshProjects")
  .addEventListener(
    "click",
    loadProjects
  );

/* =====================================================
 VOICE INPUT
===================================================== */

function setupSpeech() {
  const Recognition =
    window.SpeechRecognition ||
    window.webkitSpeechRecognition;

  if (
    !Recognition
  ) {
    toast(
      "Voice input is not supported by this browser."
    );

    return null;
  }

  const recognition =
    new Recognition();

  recognition.continuous =
    false;

  recognition.interimResults =
    true;

  recognition.lang =
    "en-IN";

  return recognition;
}

function startVoice(
  target,
  button
) {
  if (
    state.recognition
  ) {
    try {
      state.recognition.stop();
    } catch {}

    state.recognition =
      null;

    if (
      state.recordingTarget
    ) {
      state.recordingTarget
        .classList.remove(
          "recording"
        );
    }

    state.recordingTarget =
      null;

    return;
  }

  const recognition =
    setupSpeech();

  if (
    !recognition
  ) {
    return;
  }

  state.recognition =
    recognition;

  state.recordingTarget =
    button;

  button.classList.add(
    "recording"
  );

  recognition.onresult =
    event => {
      let finalText =
        "";

      for (
        let i =
          event.resultIndex;
        i <
        event.results.length;
        i++
      ) {
        finalText +=
          event.results[i][0]
            .transcript;
      }

      target.value =
        finalText.trim();
    };

  recognition.onerror =
    event => {
      toast(
        `Voice input: ${event.error}`
      );
    };

  recognition.onend =
    () => {
      button.classList.remove(
        "recording"
      );

      state.recognition =
        null;

      state.recordingTarget =
        null;
    };

  try {
    recognition.start();
  } catch (
    error
  ) {
    button.classList.remove(
      "recording"
    );

    state.recognition =
      null;

    toast(
      error.message
    );
  }
}

$("#missionMic")
  .addEventListener(
    "click",
    () =>
      startVoice(
        $("#missionInput"),
        $("#missionMic")
      )
  );

$("#globalMic")
  .addEventListener(
    "click",
    () =>
      startVoice(
        $("#missionInput"),
        $("#globalMic")
      )
  );

$("#youtubeMic")
  .addEventListener(
    "click",
    () =>
      startVoice(
        $("#youtubeSearch"),
        $("#youtubeMic")
      )
  );

/* =====================================================
 OPEN YOUTUBE LAB
===================================================== */

$("#openYoutubeLab")
  .addEventListener(
    "click",
    () => {
      const question =
        state.lastQuestion;

      showView(
        "youtube"
      );

      if (
        question
      ) {
        $("#youtubeSearch")
          .value =
          question;

        searchYouTubeUI(
          question
        );
      }
    }
  );

/* =====================================================
 INITIALIZATION
===================================================== */

async function boot() {
  loadHealth();

  loadCreatorFormats();

  loadPlaylist();

  loadProjects();

  /*
  Create player when iframe API is
  already ready.
  */

  if (
    window.YT &&
    YT.Player
  ) {
    createPlayer();
  }

  /*
  Refresh health periodically.
  */

  setInterval(
    loadHealth,
    30000
  );
}

boot();

</script>

</body>
</html>



This project is being built in public, and every suggestion helps shape the next version.ation-Workspace
A local AI workspace combining AI automation, research, YouTube intelligence, content creation, video analysis, memory, and project management in one application.
