<script setup>
import { ref, computed } from 'vue';
import axios from 'axios';

const sessionId = ref(null);
const quizCompleted = ref(false);
const currentQuestion = ref(0);
const score = ref(0);
const questions = ref([]);

const getQuestions = async () => {
  // Create a new session
  const { data } = await axios.post('http://localhost:8090/sessions', { username: 'front' });
  sessionId.value = data.sessionId;

  // // Get the session ID
  // const { data: sessionData } = await axios.get('http://localhost:8090/sessions/current/id');
  // sessionId.value = sessionData.sessionId;

  // Get the quiz questions
  const { data: quizData } = await axios.get(`http://localhost:8090/sessions/${sessionId.value}`);
  questions.value = quizData.questions;
};

const getCurrentQuestion = computed(() => {
  if (!questions.value || questions.value.length === 0) {
    return null;
  }
  if (currentQuestion.value >= questions.value.length || !questions.value[currentQuestion.value]) {
    return null; // or any other default value
  }
  const question = { ...questions.value[currentQuestion.value] };
  question.index = currentQuestion.value;
  console.log(question)
  return question;
});

const submitQuestion = () => {
  const question = questions.value[currentQuestion.value];
  if (typeof question.answer === 'boolean') { // check if question is true/false
    if ((question.selected === 'true' && question.answer) || (question.selected === 'false' && !question.answer)) { // check if answer is correct
      score.value++; // update score
    }
  } else {
    const selected = new Set(question.selected);
    const correct = new Set(question.variants.filter(v => v.is_correct).map(v => v.variant));
    if (selected.size === correct.size && [...selected].every(s => correct.has(s))) {
      score.value++; // update score
    }
  }
};

const setAnswer = evt => {
  const question = questions.value[currentQuestion.value];
  if (question.is_multiple) {
    const selected = question.selected || []; // assign to empty array if null
    const index = selected.indexOf(evt.target.value);
    if (index > -1) {
      selected.splice(index, 1);
    } else {
      selected.push(evt.target.value);
    }
    question.selected = selected; // update the question object with the new selected value
  } else {
    question.selected = evt.target.value;
  }
  evt.target.checked = false; // uncheck the checkbox or radio button
};

const nextQuestion = () => {
  if (currentQuestion.value < questions.value.length - 1) {
    currentQuestion.value++;
  } else {
    quizCompleted.value = true;
  }
};

const submitAndNextQuestion = () => {
  submitQuestion();
  nextQuestion();
};

getQuestions(); // call the function here to fetch the questions

</script>

<template>
  <main class="app" v-if="questions">
    <h1>The Quiz</h1>
    <section class="quiz" v-if="getCurrentQuestion">
      <div class="quiz-info">
        <span class="question">{{ getCurrentQuestion.text }}</span>
        <span class="score">{{ score }} / {{ questions.length }}</span>
      </div>
      <div class="question-options">
        <ul>
          <li v-for="(variant, index) in getCurrentQuestion?.variants" :key="index">
            <label>
              <input
                  :type="getCurrentQuestion?.is_multiple ? 'checkbox' : 'radio'"
                  :value="variant.variant"
                  :checked="getCurrentQuestion?.is_multiple ? getCurrentQuestion.selected?.includes(variant.variant) : getCurrentQuestion.selected === variant.variant"
                  @change="setAnswer"
              />
              {{ variant.variant }}
            </label>
          </li>
        </ul>
        <div v-if="!getCurrentQuestion.variants || getCurrentQuestion.variants.length === 0">
          <label>
            <input
                type="radio"
                value="true"
                v-model="getCurrentQuestion.selected"
                @change="setAnswer"
            />
            True
          </label>
          <label>
            <input
                type="radio"
                value="false"
                v-model="getCurrentQuestion.selected"
                @change="setAnswer"
            />
            False
          </label>
        </div>
      </div>
      <button v-if="!quizCompleted" @click="submitAndNextQuestion">Next</button>
      <div v-else>
        <h2>Quiz Completed!</h2>
        <p>Your score is {{ score }}/{{ questions.length }}</p>
      </div>
    </section>
  </main>
</template>