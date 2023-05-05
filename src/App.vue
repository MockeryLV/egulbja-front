<script setup>
import {ref, computed} from 'vue';
import axios from 'axios';

const sessionId = ref(null);
const quizCompleted = ref(false);
const currentQuestion = ref(0);
const score = ref(0);
const questions = ref([]);

const getQuestions = async (username) => {
  // Create a new session
  const {data} = await axios.post('http://localhost:8090/sessions', {username});
  sessionId.value = data.sessionId;

  // Get the quiz questions
  const {data: quizData} = await axios.get(`http://localhost:8090/sessions/${sessionId.value}`);
  questions.value = quizData.questions;
};

const getCurrentQuestion = computed(() => {
  if (!questions.value || questions.value.length === 0) {
    return null;
  }
  if (currentQuestion.value >= questions.value.length || !questions.value[currentQuestion.value]) {
    return null; // or any other default value
  }
  const question = {...questions.value[currentQuestion.value]};
  question.index = currentQuestion.value;

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
    const correct = new Set(question.variants.filter(v => v.is_correct).map(v => v.id.toString()));
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
    if (evt.target.checked) { // check if the checkbox is checked
      if (index === -1) {
        selected.push(evt.target.value);
      }
    } else {
      if (index > -1) {
        selected.splice(index, 1);
      }
    }
    question.selected = selected; // update the question object with the new selected value
  } else {
    question.selected = evt.target.value;
  }
};


const submitAnswers = async () => {
  const answers = questions.value.map((question) => {
    if (question.variants) {
      return {
        question_type: 'ma',
        question_id: question.id,
        selected_variants: [...question.selected].map((variant) => {
          return {
            variant_id: variant
          };
        }),
      };
    } else {
      return {
        question_type: 'tf',
        question_id: question.id,
        answer: question.selected === 'true' ? 1 : 0,
      };
    }
  });

  await axios.post(`http://localhost:8090/sessions/${sessionId.value}/answers`, {"answers": answers});

  quizCompleted.value = true;
};

const nextQuestion = () => {
  const currentQues = getCurrentQuestion.value;
  if (!currentQues) {
    return;
  }

  // Reset selected property
  if (currentQues.variants && currentQues.variants.length > 0) {
    currentQues.selected = [];
  } else {
    currentQues.selected = null;
  }

  if (currentQuestion.value < questions.value.length - 1) {
    currentQuestion.value++;
    uncheckOptions();
  } else {
    submitAnswers();
  }
};

const hasSelectedOption = computed(() => {
  const currentQues = getCurrentQuestion.value;
  if (!currentQues) {
    return false;
  }
  if (currentQues.variants && currentQues.variants.length > 0) {
    return currentQues.selected && currentQues.selected.length > 0;
  } else {
    return currentQues.selected !== null;
  }
});

const uncheckOptions = () => {
  const inputs = document.querySelectorAll('input[type=checkbox], input[type=radio]');
  inputs.forEach(input => {
    input.checked = false;
  });
};

const submitAndNextQuestion = () => {
  submitQuestion();
  nextQuestion();
};

const username = window.prompt('Please enter your nickname:');
getQuestions(username);

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
                  :value="variant.id"
                  :name="getCurrentQuestion?.is_multiple ? null : 'answer'"
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
                name="answer"
                @click="setAnswer"
            />
            True
          </label>
          <label>
            <input
                type="radio"
                value="false"
                name="answer"
                @click="setAnswer"
            />
            False
          </label>
        </div>
      </div>
      <button v-if="!quizCompleted" :disabled="!hasSelectedOption" @click="submitAndNextQuestion">Next</button>

      <div v-else>
        <h2>Quiz Completed!</h2>
        <p>Your score is {{ score }}/{{ questions.length }}</p>
      </div>
    </section>
  </main>
</template>