When we generate content, we publish an event with the tokens, etc. We create a module in CloudAPI that takes the tokens and updates information based on consumption.
We can make decisions based on this.

RESPONSE: {
  "id": "chatcmpl-c3e03dfc-f1a9-43f8-af4e-0eae36fb5952",
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "{\n  \"model\": \"eivolet\",\n  \"metadata\": {\n    \"name\": \"french_a1_intros\"\n  },\n  \"tags\": [\n    \"french\"\n  ],\n  \"labels\": [\n    {\n      \"culture\": \"en-US\",\n      \"title\": \"French Intro A1\",\n      \"overview\": \"An introductory French library for absolute and false beginners covering phonetics, basic conjugation, and daily conversation.\",\n      \"tooltip\": \"Beginner French courses (A1)\",\n      \"tags\": [\n        \"beginner\",\n        \"phonetics\",\n        \"conversation\"\n      ]\n    },\n    {\n      \"culture\": \"fr\",\n      \"title\": \"Français Initiation A1\",\n      \"overview\": \"Bibliothèque d'apprentissage du français pour débutants absolus et faux débutants, incluant phonétique, conjugaison de base et conversation quotidienne.\",\n      \"tooltip\": \"Cours de français niveau A1\",\n      \"tags\": [\n        \"débutant\",\n        \"phonétique\",\n        \"conversation\"\n      ]\n    },\n    {\n      \"culture\": \"es\",\n      \"title\": \"Francés Intro A1\",\n      \"overview\": \"Biblioteca introductoria de francés para principiantes absolutos y falsos principiantes, abarcando fonética, conjugación básica y conversación diaria.\",\n      \"tooltip\": \"Cursos de francés nivel A1\",\n      \"tags\": [\n        \"principiante\",\n        \"fonética\",\n        \"conversación\"\n      ]\n    }\n  ]\n}",
        "reasoning": "We need to output YAML matching the schema. Ensure model: \"eivolet\". metadata.name meets constraints: 10-30 chars, lowercase, underscores. Something like french_a1_intros (length: french(6)+_a1(3)+_intros(6) = 15) good. Must include extra.traits with language, level, method.\n\ntags: [\"french\"].\n\nlabels: exactly 3 entries: en-US, fr, es.\n\nEach label includes tags array.\n\nMake sure correct structure.\n\nYAML:\n\nmodel: \"eivolet\"\nmetadata:\n  name: french_a1_intros\n  extra:\n    traits:\n      language: french\n      level: a1\n      method: communicative\ntags: [\"french\"]\nlabels:\n  - culture: en-US\n    title: French Intro A1\n    overview: An introductory French library for absolute and false beginners covering phonetics, basic conjugation, and daily conversation.\n    tooltip: Beginner French courses (A1)\n    tags: [\"beginner\", \"phonetics\", \"conversation\"]\n  - culture: fr\n    title: Français Initiation A1\n    overview: Bibliothèque d'apprentissage du français pour débutants absolus et faux débutants, incluant phonétique, conjugaison de base et conversation quotidienne.\n    tooltip: Cours de français niveau A1\n    tags: [\"débutant\", \"phonétique\", \"conversation\"]\n  - culture: es\n    title: Francés Intro A1\n    overview: Biblioteca introductoria de francés para principiantes absolutos y falsos principiantes, abarcando fonética, conjugación básica y conversación diaria.\n    tooltip: Cursos de francés nivel A1\n    tags: [\"principiante\", \"fonética\", \"conversación\"]\n\nCheck length name: french_a1_intros = 15 characters, ok. Ensure tags array correct.\n\nReturn only YAML.",
        "role": "assistant"
      }
    }
  ],
  "created": 1772639347,
  "model": "gpt-oss-120b",
  "system_fingerprint": "fp_2d389d34367dd22b92f3",
  "object": "chat.completion",
  "usage": {
    "total_tokens": 3441,
    "completion_tokens": 720,
    "completion_tokens_details": {
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0,
      "reasoning_tokens": 0
    },
    "prompt_tokens": 2721,
    "prompt_tokens_details": {
      "cached_tokens": 0
    }
  },
  "time_info": {
    "queue_time": 2.0348754689999997,
    "prompt_time": 0.084756046,
    "completion_time": 0.253023393,
    "total_time": 2.390174150466919,
    "created": 1772639347.6643467
  }
}