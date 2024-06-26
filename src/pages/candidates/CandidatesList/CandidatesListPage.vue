<template>
  <section class="candidates">
    <div class="candidates__title">
      <TopSquareButton
        class="candidates__back-btn"
        :icon="iconBack"
        @click="$router.push({ name: 'vacanciesList' })"
      >
      </TopSquareButton>
      <h1>Кандидаты вакансии</h1>
    </div>

    <div class="candidates__filter">
      <div v-if="!vacancyId">Выберите вакансию:</div>
      <div class="candidates__filter-title">
        <div v-if="vacancyId && dataFetched">Вакансия:</div>
        <SelectMain
          v-if="dataFetched"
          v-model="vacancyId"
          :options="vacanciesIds"
          @update:modelValue="updateVacancyId"
        />
      </div>
      <div class="candidates__filter-title">
        <div v-if="vacancyId && dataFetched">Статус:</div>
        <SelectMain
          v-if="vacancyId !== '' && dataFetched"
          v-model="status"
          :options="candidateStatus"
          @update:modelValue="updateStatus"
        />
      </div>
    </div>

    <div class="candidates__description" v-if="vacancyId !== ''">
      <span>{{
        vacanciesIds.find((vacancy) => vacancy.id === vacancyId)?.name
      }}</span>
    </div>

    <div class="candidates__box-candidates">
      <!-- Отображение прелоадера  -->
      <SpinnerMain v-if="isLoading" style="width: 50px" />
      <span v-else-if="vacancyId && candidates.length === 0"
        >Кандидатов пока нет</span
      >
      <CandidateCard
        v-if="vacancyId && !isLoading"
        v-for="candidate in candidates"
        :key="candidate.candidateId"
        :candidate="candidate"
        :vacancy-id="vacancyId"
        :status="status"
        :respond="candidate.otklikId"
      />
    </div>
    <!-- Встраивание элемента в DOM дерево -->
    <Teleport to="body">
      <!-- Вывод сообщения о ошибке -->
      <ErrorNotification v-if="errorMessage" :message="errorMessage" />
    </Teleport>
  </section>
</template>

<script setup>
import { onMounted, ref, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { isManager } from '@/js/AuthFunctions';
import { MainRequestClass } from '@/js/RootClasses';
import CandidateCard from './components/CandidateCard.vue';
import ErrorNotification from '@/components/ErrorNotification.vue';
import SelectMain from '@/components/SelectMain.vue';
import SpinnerMain from '@/components/SpinnerMain.vue';
import TopSquareButton from '@/components/TopSquareButton.vue';
import iconBack from '@/assets/icons/back.svg';

const router = useRouter();
const route = useRoute();

//Проверка авторизации пользователя
if (!isManager()) router.push({ name: 'home' });

//Ref-переменные
const vacanciesIds = ref([]); // список id вакансий
const candidates = ref([]); // список кандидатов
const vacancyId = ref(''); // ID вакансии
const status = ref('Все'); // Статус кандидата
const candidateStatus = ref([{ name: 'Все', id: 'Все' }]); // Статусы кандидатов

//Флаги загрузки данных
const dataFetched = ref(false);
const isLoading = ref(false);

// Отображение ошибки
const errorMessage = ref('');

// Получение из роута параметров, если есть query
if (route.query.vacancyId) {
  vacancyId.value = route.query.vacancyId;
  if (route.query.status) {
    status.value = route.query.status;
  }
}

//Получение списка id вакансий при загрузке страницы
function getAllVacanciesManager() {
  let requestClass = new MainRequestClass();

  requestClass.request(
    '/vacancies/get_all_vacancies.php',
    'manager',
    function (response) {
      //успешный результат

      //Заполнение массива id и названия вакансий
      response.vacancies.map((vacancy) => {
        vacanciesIds.value.push({
          name: `${vacancy.name}(id:${vacancy.id})`,
          id: vacancy.id,
        });
      });

      // Установление флага загрузки данных
      dataFetched.value = true;
    },
    function (err) {
      //неуспешный результат
      errorMessage.value = err;
    }
  );
}

//Получение списка кандидатов по id вакансии
function getAllCandidatesManager() {
  class CandidatesGetCandidatesByVacancyId extends MainRequestClass {
    vacancyId = vacancyId.value; // ID вакансии, отклики для которой нужно получить
    status = status.value === 'Все' ? '' : status.value; // статус отклика, по которому нужно получить отклики (если пустая строка, то этот фильтр не учитывается)
  }
  let requestClass = new CandidatesGetCandidatesByVacancyId();

  // Установление флага прелоадера
  isLoading.value = true;

  //Получение списка кандидатов
  if (vacancyId.value) {
    requestClass.request(
      '/candidates/get_candidates_by_vacancy_id.php',
      'manager',
      function (response) {
        //успешный результат
        candidates.value = response.candidates;
        isLoading.value = false;
      },
      function (err) {
        //неуспешный результат
        errorMessage.value = err;
        isLoading.value = false;
      }
    );
  }
}

// Получение статусов кандидатов к вакансии
function getVacancyStatuses() {
  class VacanciesGetVacanciesStatuses extends MainRequestClass {
    vacancyId = vacancyId.value; // ID вакансии, для которой нужно получить доступные статусы
  }
  let requestClass = new VacanciesGetVacanciesStatuses();

  //Получение статусов
  if (vacancyId.value) {
    requestClass.request(
      '/vacancies/get_vacancy_statuses.php',
      'manager',
      function (response) {
        //успешный результат
        candidateStatus.value = [{ name: 'Все', id: 'Все' }];
        response.statuses.map((status) => {
          candidateStatus.value.push({
            name: status.statusName,
            id: status.statusComment,
          });
        });
      },
      function (err) {
        //неуспешный результат
        errorMessage.value = err;
      }
    );
  }
}

//Обработчик смены query параметра при смене id
const updateVacancyId = () => {
  router.push({
    query: { ...route.query, vacancyId: vacancyId.value, status: status.value },
  });
};

//Обработчик смены query параметра при смене id
const updateStatus = () => {
  router.push({ query: { ...route.query, status: status.value } });
};

//Получение списка кандидатов при загрузке страницы
onMounted(() => {
  getAllVacanciesManager();
  if (route.query.vacancyId) {
    getAllCandidatesManager();
    getVacancyStatuses();
  }
});

//Получение списка кандидатов и статусов при обновлении query параметров
watch(
  () => route.query.vacancyId,
  () => {
    getVacancyStatuses();
    getAllCandidatesManager();
  }
);

//Получение списка кандидатов при обновлении query status
watch(
  () => route.query.status,
  () => {
    getAllCandidatesManager();
  }
);
</script>

<style scoped>
.candidates {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-bottom: 80px;
}

.candidates__title {
  width: 100%;
  text-align: center;
}

.candidates__filter {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  gap: 30px;
}

.candidates__filter-title {
  display: flex;
  align-items: center;
  gap: 5px;
  flex-wrap: wrap;
}

.candidates__description {
  text-align: center;
  margin-top: 30px;
}

.candidates__box-candidates {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 15px;
  margin-top: 40px;
  width: 100%;
  max-width: 95%;
  :nth-child(n) {
    max-width: 700px;
  }
}

.candidates__back-btn {
  position: relative;
  top: 20px;
}

@media screen and (max-width: 350px) {
  .candidates__title {
    h1 {
      margin-top: 30px;
    }
  }
}

@media screen and (max-width: 450px) {
  .candidates__filter {
    gap: 15px;
  }
}
</style>
