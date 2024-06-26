<template>
  <div>
    <section v-if="!openAiKey">
      <form @submit.prevent="setOpenAiKey">
        <input type="text" v-model="currentOpenAiKey" placeholder="Enter OpenAI API Key to Begin" />
        <button>Save Key</button>
      </form>
      <div class="notice">Your key is stored in LocalStorage. There are no servers behind Kevin so nothing is
        transmitted.</div>
    </section>
    <article :className="isReady ? '' : 'disabled'">
      <h2>Specify a goal you&rsquo;re looking to execute</h2>
      <p><em>Note: Kevin is currently in Alpha, so please only give him small goals for now.</em></p>

      <form :className="isFormEnabled ? '' : 'disabled'" @submit.prevent="planAndExecuteGoal">
        <input type="text" v-model="goal" placeholder="Write a program that generates the fibonacci sequence" />
        <button :disabled="!isFormEnabled || !isReady">
          {{ isFormEnabled ? "Execute Goal" : "Executing..." }}
        </button>
      </form>
    </article>
  </div>
</template>

<style scoped>
section {
  text-align: center;
}

section,
section input,
section button {
  font-family: 'Computer Modern Sans', monospace;
}

.notice {
  display: inline-block;
  background-color: #ffcccc;
  border: 1px solid #ff6666;
  border-radius: .25em;
  padding: .375em .5em .25em .5em;
  color: #333;
  font-size: 0.875em;
  line-height: 1;
}

section button {
  background-color: #4CAF50;
  border: none;
  color: white;
  padding: 0.5em 1em;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 1em;
  margin: 0.25em 0.5em;
  cursor: pointer;
  transition: all 0.3s ease;
  border-radius: .25em;
}

section button:hover {
  background-color: #45a049;
}

article {
  text-align: center;
}

form {
  margin: 1em auto;
  width: 100%;
}

.disabled {
  opacity: .4;
}

input {
  padding: .5em .75em;
  border: 1px solid #ccc;
  border-radius: 2px;
  display: inline-block;
  width: 30em;
  margin-right: .5em;
}

article button {
  border: 1px solid #999;
  padding: .5em .75em;
  border: none;
  border-radius: 2px;
  color: white;
  cursor: pointer;
  display: inline-block;
  background: linear-gradient(45deg, #ee7752, #e73c7e, #23a6d5, #23d5ab);
  background-size: 400% 400%;
  animation: rainbow 5s alternate infinite;
}

@keyframes rainbow {
  0% { background-position: 0% 50%; }
  100% { background-position: 100% 50%; }
}

button:active {
  background-color: navy;
}
</style>

<script>
import EventBus from '../lib/EventBus.js'

export default {
  name: 'ChatView',
  data() {
    return {
      currentOpenAiKey: '',
      openAiKey: '',
      goal: "",
      isFormEnabled: true,
      isReady: false,
      messages: [],
    };
  },
  watch: {
    openAiKey(newKey) {
      if (newKey) {
        localStorage.openAiKey = newKey
      }
      this.isReady = true
    }
  },
  mounted () {
    if (localStorage.openAiKey) {
      this.openAiKey = localStorage.openAiKey;
    }
    EventBus.on('executionComplete', this.enableForm)
    // EventBus.emit('executeLinesWhenReady', [
    //   "mkdir fibonacci_project",
    //   "cd fibonacci_project",
    //   "npm init -y",
    //   "npm install --save jest",
    //   "nano fibonacci.js",
    //   "function fibonacci(n) {",
    //   "  let fib = [0, 1];",
    //   "  for (let i = 2; i <= n; i++) {",
    //   "    fib[i] = fib[i - 1] + fib[i - 2];",
    //   "  }",
    //   "  return fib[n];",
    //   "}",
    //   "module.exports = fibonacci;",
    //   "\u0018",
    //   "Y",
    //   "nano fibonacci.test.js",
    //   "const fibonacci = require('./fibonacci');",
    //   "test('fibonacci sequence element at position 5 to equal 5', () => {",
    //   "  expect(fibonacci(5)).toBe(5);",
    //   "});",
    //   "\u0018",
    //   "Y",
    //   "npx jest"
    // ])
  },
  methods: {
    disableForm() {
      this.isFormEnabled = false
      this.goal = ""
    },
    enableForm() {
      this.isFormEnabled = true
    },
    setOpenAiKey() {
      this.isReady = true
      this.openAiKey = this.currentOpenAiKey
    },
    chat(message) {
      this.messages.push({
        role: "user",
        content: message
      }); // Append the new message to the messages array
      console.log(message)
      // Send the entire messages array to the OpenAI API
      return fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.openAiKey}`
        },
        body: JSON.stringify({
          model: 'gpt-4-turbo-preview',
          messages: this.messages
        }),
      })
        .then((response) => response.json())
        .then((data) => {
          if (data.error) throw new Error(data.error.message);
          console.log(data)
          // Update the this.messages array with the response from OpenAI
          const choices = data.choices;
          const lastChoice = choices[choices.length - 1];
          const responseMessage = lastChoice.message;
          this.messages.push({
            role: "assistant",
            content: responseMessage.content
          });
          console.log(this.messages)
          return responseMessage.content
        })
        .catch((error) => {
          console.error(error);
        });
    },
    async planAndExecuteGoal () {
      const goal = this.goal.toLowerCase()
      this.disableForm()
      const message = `
You are inside a shell.
Provide a series of terminal commands one after the other to ${goal}

Use only valid node js code and jest for testing.
Also write a test for the same.

All the text you give will directly be sent to a terminal, so don't include any unnecessary text and always give full response for any code. ONLY reply in valid terminal commands. Do NOT use echo or sed, only use "nano" to edit files.

EVERYTHING YOU SAY WILL DIRECTLY BE SENT TO A TERMINAL, so only reply with a stream of keypresses that will get the result. Please only provide a stream of keypresses to be piped into an actual terminal without any explanation or text. You can provide control characters like ^X to close nano. Instead of "Enter", use \\n. Please only send valid commands and keypresses that can execute the goal in a terminal directly, do not write any explanation or comments or I might lose my job.`
      const responseMessage = await this.chat(message)

      // remove ```bash and ``` from the message
      let responseMessageCleaned = responseMessage
        .replace(/```(shell|sh|bash)/g, '')
        .replace(/```/g, '')

      // convert all escape sequences to ascii characters. For example, ^O is represented by \x0f (ASCII 15) and ^X is represented by \x18 (ASCII 24).
      responseMessageCleaned = responseMessageCleaned
        .replace(/\^O/g, '\x0f')
        .replace(/\^X/g, '\x18')

      // replace \\n to \n
      responseMessageCleaned = responseMessageCleaned.replace(/\\n/g, '\n')

      // Send the response to the socket, line by line, using executeLikeNaturalTyping
      const lines = responseMessageCleaned
        .split('\n')
        .filter(line => line != '')
      console.log(lines)
      EventBus.emit('executeLinesWhenReady', lines)
    },
  },
};
</script>