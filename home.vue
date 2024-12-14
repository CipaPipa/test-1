<template>
  <div class="home">
    <div class="home__last-update">Последнее обновление: {{ lastUpdateTime }}</div>
    <table>
      <thead>
        <tr>
          <th>
            <div class="created">
              <div>Дата создания | Крайний срок</div>
              <div class="created__sort"
                :class="{ 'created__sort--asc': sort === 'asc', 'created__sort--desc': sort === 'desc' }"
                @click="sortByDate(true)">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 16 16" fill="none">
                  <path
                    d="M5.55352 7.27011L7.47725 9.19384C7.76693 9.48351 8.23486 9.48351 8.52454 9.19384L10.4483 7.27011C10.9162 6.80217 10.582 6 9.92091 6H6.07345C5.4124 6 5.08559 6.80217 5.55352 7.27011Z"
                    fill="#61656A" />
                </svg>
              </div>
            </div>
          </th>
          <th>
            <form>
              <select @change="filterBy" name="user" id="user-select">
                <option selected>Все</option>
                <option :value="user" v-for="user in users" :key="user"> {{ user }} </option>
              </select>
            </form>
          </th>
          <th>Наименование | Описание</th>
          <th>Номер</th>
          <th>Инициатор</th>
          <th>Текущий этап</th>
        </tr>
      </thead>
      <tbody>
        <tr v-show="task.visible" v-for="(task, index) in tasks" :key="index">
          <td>
            <div class="status" :class="[`${getColor(task['ДатаСоздания'], task['КрайнийСрокУстранения']).status}`]">
              <div class="status__circle"></div>
              <div class="status__text">{{ getColor(task['ДатаСоздания'], task['КрайнийСрокУстранения']).text }}</div>
            </div>
            <div class="time">{{ new Date(task['ДатаСоздания']).toLocaleDateString('ru-RU') }} | {{
      new Date(task['КрайнийСрокУстранения']).toLocaleDateString('ru-RU') }}</div>
          </td>
          <td>
            <div class="user">
              <div class="user__icon" :style="`background-color: ${task.color}`"></div>
              <div class="user__name">{{ task['ТекущийИсполнитель'] }}</div>
            </div>
          </td>
          <td>
            <button class="popover-button" popovertarget="modal" @click="activeTask = task">{{ task['Наименование']
              }}</button>
          </td>
          <td>Запрос {{ task['Number'] }}</td>
          <td>{{ task['Инициатор'] }}</td>
          <td>{{ task['ТекущийЭтап'] }} ({{ task['ТекущийЭтап_PN'] }})</td>
        </tr>
      </tbody>
    </table>


    <div id="modal" popover>
      <div class="modal" v-if="activeTask">
        <div class="modal__title">{{ activeTask['Наименование'] }}</div>
        <div class="modal__text">{{ activeTask['Описание'] }}</div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "Home",
  data() {
    return {
      users: [],
      tasks: [],
      sort: 'desc',
      lastUpdateTime: '',
      activeTask: null
    }
  },
  mounted() {
    this.loadTasks();
  },
  methods: {
    loadTasks() {
      this.lastUpdateTime = new Date().toLocaleDateString('ru-RU', { hour: '2-digit', minute: '2-digit', second: '2-digit', day: '2-digit', month: '2-digit', year: 'numeric' });
      if (localStorage.getItem('tasks')) {
        this.tasks = JSON.parse(localStorage.getItem('tasks'));
        const usersSet = new Set();
        this.tasks.forEach((task) => {
          task.visible = true;
          usersSet.add(task['ТекущийИсполнитель']);
        });
        this.users = Array.from(usersSet);
        this.generateColors();
      } else {
        if (!this.tasks.length) {
          axios.get("http://internal.pa.local/itil_test/zayavki.php").then((response) => {
            this.tasks = response.data;
            this.sortByDate(false);
            const usersSet = new Set();
            this.tasks.forEach((task) => {
              task.visible = true;
              usersSet.add(task['ТекущийИсполнитель']);
            });
            this.users = Array.from(usersSet);
            this.generateColors();

            localStorage.setItem('tasks', JSON.stringify(this.tasks));
          });
        }
      }
      console.log(this.users);
      console.log(this.tasks);
    },
    sortByDate(changeSort) {
      if (changeSort) {
        if (this.sort === 'asc') {
          this.sort = 'desc'
        } else {
          this.sort = 'asc'
        }
      }

      if (this.sort === 'asc') {
        this.tasks.sort((a, b) => {
          return new Date(a['ДатаСоздания']) - new Date(b['ДатаСоздания'])
        })
      } else if (this.sort === 'desc') {
        this.tasks.sort((a, b) => {
          return new Date(b['ДатаСоздания']) - new Date(a['ДатаСоздания'])
        })
      } else {
        this.tasks.sort((a, b) => {
          return b['ТекущийИсполнитель'].replaceAll(/\s/, '').toLowerCase() - a['ТекущийИсполнитель'].replaceAll(/\s/, '').toLowerCase()
        })
      }
    },
    filterBy(event) {
      this.tasks.forEach((task) => {
        if (event.target.value === 'Все') {
          this.tasks.forEach((task) => {
            task.visible = true;
          })
        } else if (task['ТекущийИсполнитель'] !== event.target.value) {
          task.visible = false;
        } else {
          task.visible = true;
        }
      })
    },
    getColor(creationDate, expirationDate) {
      const now = new Date();
      const creation = new Date(creationDate);
      const expiration = new Date(expirationDate);

      const unzeronedCreation = new Date(creationDate);
      const unzeronedExpiration = new Date(expirationDate);

      const toZeroHours = (date) => date.setHours(0, 0, 0, 0);

      toZeroHours(creation);
      toZeroHours(expiration);

      const dateInDays = (date) => Math.ceil(date / (1000 * 60 * 60 * 24));
      const dateInHours = (date) => Math.ceil(date / (1000 * 60 * 60));

      const daysSinceCreation = dateInDays(now - creation);
      const daysUntilExpiration = dateInDays(expiration - now);

      if (daysUntilExpiration < 0) {
        return { status: 'status--red', text: `Просрочено на ${daysUntilExpiration * -1} дн.` };
      } else if (daysUntilExpiration > 0 && daysUntilExpiration < 7) {
        return { status: 'status--orange', text: `Истекает через ${daysUntilExpiration} дн.` };
      } else if (daysUntilExpiration > 7 && daysUntilExpiration < 14) {
        return { status: 'status--orange', text: `Истекает через ${daysUntilExpiration} дн.` };
      } else {
        return { status: 'status--green', text: `Прошло ${daysSinceCreation} дн.` };
      }
    },
    generateColors() {
      const generatedColors = {};

      this.users.forEach((user) => {
        let color = this.generateRandomHexColor();
        generatedColors[user] = color;
        while (Object.values(generatedColors).filter(el => el === color).length > 1) {
          color = this.generateRandomHexColor();
          generatedColors[user] = color;
        }
      })
      this.tasks.forEach((task) => {
        if (!task.color) {
          task.color = generatedColors[task['ТекущийИсполнитель']];
        }
      })
    },
    generateRandomHexColor() {
      const r = Math.floor(Math.random() * 256).toString(16).padStart(2, '0');
      const g = Math.floor(Math.random() * 256).toString(16).padStart(2, '0');
      const b = Math.floor(Math.random() * 256).toString(16).padStart(2, '0');
      return `#${r}${g}${b}`;
    }
  }
}
</script>

<style lang="scss" scoped></style>