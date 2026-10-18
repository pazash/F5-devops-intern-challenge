# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Gemini

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. `i have an error in my docker compose configuration.

this is my current docker compose yml:

services:

  backend:

    build:

      context: ./backend

    # Published for local debugging: curl http://localhost:8080/health

    ports:

      - "8080:8080"

    environment:

      - PORT=8080



  proxy:

    build:

      context: ./proxy

    # Published for local debugging: curl http://localhost:8081/nginx-health

    ports:

      - "8081:80"

    depends_on:

      - backend



  test-runner:

    build:

      context: ./test-runner

    depends_on:

      - proxy

    environment:

      - PROXY_URL=http://proxy:80





the error is that test runner runs before the backend container starts listening and fails because he cant get an answer from the backend.



please make a research and tell me what are the ways to make the test-runner wait until the backend starts listening`
2. `I am writing a dockerfile for a minimal nginx server that will act as reverse proxy.

please search online and docker hub and research what should be the minimal and optimal base image for such server.

I saw online alpine and scratch are minimal base image.

What is the optimal one?`
3. `give me a way to make sure that the container runs as non root user. is there a command to see which user is the docker container runs from?`

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> I asked the AI for researching what should be the changes in the configuration to fit for non-root user.
I knew that the base image I used supposed to run as a non-root user but wanted to make sure.
After stating correctly that the base image runs as non-root the AI stated I need to change some folder configurations in nginx.conf to not write to folder it doesn't have privilages to as non root user and that otherwise the container would crash.
But the changes was unnecessary because I already checked and the proxy server ran just fine.
So I ignored his changes to not insert complex and unknown configuration changes.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.
nothing
> ...
