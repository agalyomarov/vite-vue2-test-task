<script lang="ts">
import { ref, reactive, computed, onMounted, watch } from "vue";
import axios from "axios";
export default {
  setup() {
    const apiUrl = "http://localhost:8000";

    const users = ref<any[]>([]);
    const projects = ref<any[]>([]);
    const newProject = reactive({ name: "", description: "", user_id: null });
    const taskForms = reactive<
      Record<number, { name: string; description: string; user_id: number }>
    >({});

    watch(
      () => newProject.user_id,
      () => {
        fetchProjects();
      }
    );

    // Загрузка проектов
    const fetchUsers = async () => {
      const res = await axios.get(`${apiUrl}/api/users`);
      users.value = res.data.data;
    };
    // Загрузка проектов
    const fetchProjects = async () => {
      const res = await axios.get(`${apiUrl}/api/user/${newProject.user_id}/projects`);
      projects.value = res.data.data;
      for (const project of projects.value) {
        const res = await axios.get(`${apiUrl}/api/projects/${project.id}`);
        project.tasks = res.data.data.tasks;
        taskForms[project.id] = {
          name: "",
          description: "",
          user_id: project.user_id,
        };
      }
    };

    // Создание проекта
    const createProject = async () => {
      const res = await axios.post(`${apiUrl}/api/projects`, {
        user_id: newProject.user_id,
        name: newProject.name,
        description: newProject.description,
      });
      const newProj = res.data.data;

      const taskRes = await axios.get(`${apiUrl}/api/projects/${newProj.id}`);
      newProj.tasks = taskRes.data.data.tasks;
      projects.value.unshift(newProj);
      taskForms[newProj.id] = { name: "", description: "" };
      newProject.name = "";
      newProject.description = "";
    };

    // Удаление задачи
    const deleteTask = async (projectId: number, taskId: number) => {
      await axios.delete(`${apiUrl}/api/projects/${projectId}/tasks/${taskId}`);
      const project = projects.value.find((p) => p.id === projectId);
      if (project) {
        const res = await axios.get(`${apiUrl}/api/projects/${projectId}`);
        project.tasks = res.data.data.tasks;
      }
    };

    // Создание задачи
    const createTask = async (projectId: number) => {
      const form = taskForms[projectId];
      await axios.post(`${apiUrl}/api/projects/${projectId}/tasks`, {
        user_id: form.user_id,
        name: form.name,
        description: form.description,
      });
      const project = projects.value.find((p) => p.id === projectId);
      if (project) {
        const res = await axios.get(`${apiUrl}/api/projects/${projectId}`);
        project.tasks = res.data.data.tasks;
      }
      form.name = "";
      form.description = "";
    };

    // Последние 3 задачи по created_at
    const latestTasks = computed(() => {
      return projects.value
        .flatMap((p) => p.tasks)
        .sort(
          (a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
        )
        .slice(0, 3);
    });

    onMounted(async () => {
      await fetchUsers();
      newProject.user_id = users.value[0].id;
      fetchProjects();
    });
    return {
      createProject,
      deleteTask,
      createTask,
      users,
      projects,
      newProject,
      taskForms,
      latestTasks,
    };
  },
};
</script>

<template>
  <div>
    <form @submit.prevent="createProject">
      <h1>Проекты</h1>
      <label>
        <span>Пользователь</span>
        <select name="user_id" require v-model="newProject.user_id">
          <option disabled value="" selected>Выберите пользователя</option>
          <option v-for="user in users" :key="user.id" :value="user.id">
            {{ user.name }}
          </option>
        </select>
        <br />
        <br />
        <label>
          <span>Название</span>
          <input v-model="newProject.name" type="text" required />
        </label>
        <br />
        <br />
        <label>
          <span>Описание</span>
          <textarea v-model="newProject.description"></textarea>
        </label>
        <button type="submit">Создать</button>
      </label>
    </form>
    <br />
    <br />

    <ul style="display: flex; flex-direction: column; gap: 20px">
      <li v-for="project in projects" :key="project.id">
        <div style="display: flex; flex-direction: row; gap: 40px">
          <div>
            <div>Project Name: {{ project.name }}</div>
            <div>
              <ul>
                <li v-for="task in project.tasks" :key="task.id">
                  <div>
                    <div style="font-weight: 700">task name: {{ task.name }}</div>
                    <div style="font-size: 12px">
                      task description: {{ task.description }}
                    </div>
                  </div>
                </li>
              </ul>
            </div>
            <form @submit.prevent="createTask(project.id)">
              <input
                placeholder="task name"
                type="text"
                v-model="taskForms[project.id].name"
              />
              <br />
              <textarea
                v-model="taskForms[project.id].description"
                placeholder="task description"
              ></textarea>
              <br />
              <button @click="">Создать</button>
            </form>
          </div>
          <div v-if="project.description">Description: {{ project.description }}</div>
        </div>
      </li>
    </ul>
  </div>
</template>
