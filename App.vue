<template>
  <div :class="['container', isDark ? 'dark' : '']">
    <h1>📝 Task Tracker</h1>

    <!-- Theme Toggle -->
    <button class="theme-toggle" @click="isDark = !isDark">
      {{ isDark ? '☀️ Light Mode' : '🌙 Dark Mode' }}
    </button>

    <!-- Stats -->
    <div class="stats">
      <span>Total: {{ tasks.length }}</span>
      <span>✔ Done: {{ tasks.filter(t => t.completed).length }}</span>
      <span>❗ To Do: {{ tasks.filter(t => !t.completed).length }}</span>
    </div>

    <!-- Filter Buttons -->
    <div class="filters">
      <button @click="filter = 'all'" :class="{ active: filter === 'all' }">All</button>
      <button @click="filter = 'completed'" :class="{ active: filter === 'completed' }">Completed</button>
      <button @click="filter = 'incomplete'" :class="{ active: filter === 'incomplete' }">Incomplete</button>
    </div>

    <!-- Add Task -->
    <form @submit.prevent="addTask" class="form">
      <input v-model="newTask" placeholder="Add new task..." class="input" />
      <button class="button">Add</button>
    </form>

    <!-- Draggable Task List -->
    <draggable v-model="tasks" item-key="id" class="task-list" :animation="200">
      <template #item="{ element: task }">
        <li class="task">
          <div>
            <span
              v-if="!task.editing"
              :class="{ completed: task.completed }"
              @click="toggleComplete(task)"
            >
              {{ task.text }}
            </span>
            <input
              v-else
              v-model="task.editText"
              class="edit-input"
              @keyup.enter="saveEdit(task)"
            />
            <div class="timestamp">
              {{
                task.createdAt.toDate
                  ? task.createdAt.toDate().toLocaleString()
                  : new Date(task.createdAt).toLocaleString()
              }}
            </div>
          </div>
          <div>
            <button @click="startEditing(task)">✏️</button>
            <button @click="deleteTask(task)">🗑</button>
          </div>
        </li>
      </template>
    </draggable>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import { db } from "./firebase";
import {
  collection,
  addDoc,
  doc,
  updateDoc,
  deleteDoc,
  onSnapshot,
  query,
  orderBy,
} from "firebase/firestore";
import draggable from "vuedraggable";

// Reactive state
const newTask = ref("");
const tasks = ref([]);
const filter = ref("all");
const isDark = ref(false);

const tasksCollection = collection(db, "tasks");

// Real-time fetch tasks ordered by createdAt
onMounted(() => {
  const q = query(tasksCollection, orderBy("createdAt"));
  onSnapshot(q, (querySnapshot) => {
    tasks.value = [];
    querySnapshot.forEach((docSnap) => {
      tasks.value.push({ id: docSnap.id, ...docSnap.data() });
    });
  });
});

// Filtered tasks computed property
const filteredTasks = computed(() => {
  if (filter.value === "completed")
    return tasks.value.filter((t) => t.completed);
  if (filter.value === "incomplete")
    return tasks.value.filter((t) => !t.completed);
  return tasks.value;
});

// Add a new task to Firestore
async function addTask() {
  if (newTask.value.trim() === "") return;
  await addDoc(tasksCollection, {
    text: newTask.value,
    completed: false,
    createdAt: new Date(),
    editing: false,
  });
  newTask.value = "";
}

// Toggle task completion status
async function toggleComplete(task) {
  const taskDoc = doc(db, "tasks", task.id);
  await updateDoc(taskDoc, { completed: !task.completed });
}

// Delete a task
async function deleteTask(task) {
  const taskDoc = doc(db, "tasks", task.id);
  await deleteDoc(taskDoc);
}

// Start editing a task
function startEditing(task) {
  task.editing = true;
  task.editText = task.text;
}

// Save edited task text to Firestore
async function saveEdit(task) {
  if (task.editText.trim() !== "") {
    const taskDoc = doc(db, "tasks", task.id);
    await updateDoc(taskDoc, { text: task.editText });
    task.editing = false;
  }
}
</script>

<style scoped>
.container {
  max-width: 600px;
  margin: 40px auto;
  padding: 20px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  font-family: "Segoe UI", sans-serif;
  transition: 0.3s;
}

.dark {
  background: #1e1e1e;
  color: white;
}

.theme-toggle {
  float: right;
  background: none;
  border: none;
  color: inherit;
  cursor: pointer;
  font-size: 14px;
  margin-bottom: 10px;
}

.stats,
.filters {
  display: flex;
  justify-content: space-between;
  margin-bottom: 10px;
}

.filters button {
  padding: 6px 12px;
  border: none;
  background: #eee;
  border-radius: 6px;
  cursor: pointer;
}

.filters .active {
  background: #007bff;
  color: white;
}

.form {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.input {
  flex: 1;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid #ccc;
}

.button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}

.button:hover {
  background-color: #0056b3;
}

.task-list {
  list-style: none;
  padding: 0;
}

.task {
  background: #f9f9f9;
  padding: 12px;
  margin-bottom: 10px;
  border-radius: 8px;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-wrap: wrap;
}

.dark .task {
  background: #2e2e2e;
}

.completed {
  text-decoration: line-through;
  color: gray;
  cursor: pointer;
}

.task span,
.task input {
  cursor: pointer;
  font-size: 16px;
}

.task button {
  background: none;
  border: none;
  color: #007bff;
  cursor: pointer;
  font-size: 18px;
  margin-left: 5px;
}

.task button:hover {
  color: red;
}

.timestamp {
  font-size: 12px;
  color: #888;
  margin-top: 4px;
}

.edit-input {
  padding: 5px;
  border-radius: 6px;
  border: 1px solid #ccc;
}
</style>

