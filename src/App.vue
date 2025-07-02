<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

interface Todo {
  id: number;
  text: string;
  completed: boolean;
  isEditing?: boolean; // Optional property for editing state
  editedText?: string; // Optional property to hold the text being edited
}

// Helper to format date to YYYY-MM-DD string
const formatDate = (date: Date): string => {
  const year = date.getFullYear();
  const month = (date.getMonth() + 1).toString().padStart(2, '0');
  const day = date.getDate().toString().padStart(2, '0');
  return `${year}-${month}-${day}`;
};

// Helper to add/subtract days
const addDays = (date: Date, days: number): Date => {
  const newDate = new Date(date);
  newDate.setDate(date.getDate() + days);
  return newDate;
};

const currentDate = ref<Date>(new Date()); // Stores the currently viewed date
const formattedCurrentDate = computed(() => formatDate(currentDate.value));

// Todos are now stored per date
const todosByDate = ref<Record<string, Todo[]>>({}); // { 'YYYY-MM-DD': [{id, text, completed}] }
const newTodoText = ref('');
let nextTodoId = 0; // Global unique ID for todos

// Computed property for todos of the current date
const currentDayTodos = computed(() => {
  return todosByDate.value[formattedCurrentDate.value] || [];
});

// New computed properties for filtering todos
const pendingTodos = computed(() => {
  return currentDayTodos.value.filter(todo => !todo.completed);
});

const completedTodos = computed(() => {
  return currentDayTodos.value.filter(todo => todo.completed);
});

// Drag and Drop functionality
const draggedTodo = ref<Todo | null>(null);
const draggedOverTodo = ref<Todo | null>(null);

const handleDragStart = (todo: Todo) => {
  draggedTodo.value = todo;
};

const handleDragOver = (todo: Todo) => {
  if (draggedTodo.value && draggedTodo.value !== todo) {
    draggedOverTodo.value = todo;
  }
};

const handleDrop = (targetTodo: Todo) => {
  if (draggedTodo.value && draggedTodo.value !== targetTodo) {
    const todos = todosByDate.value[formattedCurrentDate.value];
    if (todos) {
      const fromIndex = todos.indexOf(draggedTodo.value);
      const toIndex = todos.indexOf(targetTodo);

      if (fromIndex !== -1 && toIndex !== -1) {
        const [removed] = todos.splice(fromIndex, 1);
        todos.splice(toIndex, 0, removed);
      }
    }
  }
  draggedOverTodo.value = null;
};

const handleDragLeave = (todo: Todo) => {
  if (draggedOverTodo.value === todo) {
    draggedOverTodo.value = null;
  }
};

const handleDragEnd = () => {
  draggedTodo.value = null;
  draggedOverTodo.value = null;
};

// Load todos from local storage on component mount
onMounted(() => {
  const savedTodos = localStorage.getItem('todosByDate');
  if (savedTodos) {
    todosByDate.value = JSON.parse(savedTodos);
    // Find the maximum ID across all todos to ensure uniqueness
    let maxId = -1;
    for (const dateKey in todosByDate.value) {
      todosByDate.value[dateKey].forEach(todo => {
        if (todo.id > maxId) {
          maxId = todo.id;
        }
      });
    }
    nextTodoId = maxId + 1;
  }
});

// Watch for changes in todosByDate and save to local storage
watch(todosByDate, (newTodosByDate) => {
  localStorage.setItem('todosByDate', JSON.stringify(newTodosByDate));
}, { deep: true });

const addTodo = () => {
  if (newTodoText.value.trim() === '') return;

  // Ensure the array for the current date exists
  if (!todosByDate.value[formattedCurrentDate.value]) {
    todosByDate.value[formattedCurrentDate.value] = [];
  }

  todosByDate.value[formattedCurrentDate.value].push({
    id: nextTodoId++,
    text: newTodoText.value.trim(),
    completed: false
  });
  newTodoText.value = '';
};

const editTodo = (todo: Todo) => {
  todo.isEditing = true;
  todo.editedText = todo.text;
};

const saveEdit = (todo: Todo) => {
  if (todo.editedText !== undefined) {
    todo.text = todo.editedText.trim();
    todo.isEditing = false;
    todo.editedText = undefined; // Clear editedText after saving
  }
};

const removeTodo = (todo: Todo) => {
  if (todosByDate.value[formattedCurrentDate.value]) {
    todosByDate.value[formattedCurrentDate.value] = todosByDate.value[formattedCurrentDate.value].filter((t) => t.id !== todo.id);
    // If no todos left for the day, remove the date entry
    if (todosByDate.value[formattedCurrentDate.value].length === 0) {
      delete todosByDate.value[formattedCurrentDate.value];
    }
  }
};

const moveTodoToNextDay = (todo: Todo) => {
  // Remove from current day
  removeTodo(todo);

  // Add to next day
  const nextDay = addDays(currentDate.value, 1);
  const formattedNextDay = formatDate(nextDay);

  if (!todosByDate.value[formattedNextDay]) {
    todosByDate.value[formattedNextDay] = [];
  }
  todosByDate.value[formattedNextDay].push({ ...todo, completed: false }); // Ensure it's not completed when moved
};

const remainingTodos = computed(() => {
  return pendingTodos.value.length;
});

const goToPreviousDay = () => {
  currentDate.value = addDays(currentDate.value, -1);
};

const goToNextDay = () => {
  currentDate.value = addDays(currentDate.value, 1);
};

// Handle direct date input change
const handleDateInputChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.value) {
    currentDate.value = new Date(target.value);
  }
};

const exportTodos = () => {
  let exportContent = "My Todo List\n\n";

  // Sort dates to ensure consistent output
  const sortedDates = Object.keys(todosByDate.value).sort();

  sortedDates.forEach(date => {
    const todos = todosByDate.value[date];
    if (todos.length > 0) {
      exportContent += `--- ${date} ---\n`;
      todos.forEach(todo => {
        const status = todo.completed ? "[x]" : "[ ]";
        exportContent += `${status} ${todo.text}\n`;
      });
      exportContent += "\n";
    }
  });

  const blob = new Blob([exportContent], { type: "text/plain;charset=utf-8" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = "todo-list.txt";
  link.click();
  URL.revokeObjectURL(link.href);
};
</script>

<template>
  <div class="todo-app">
    <h1 @click="exportTodos" style="cursor: pointer;">My Daily Todos</h1>

    <div class="date-navigation">
      <button @click="goToPreviousDay" class="nav-button">&lt;</button>
      <input type="date" :value="formattedCurrentDate" @change="handleDateInputChange" class="date-input" />
      <button @click="goToNextDay" class="nav-button">&gt;</button>
    </div>

    <div class="input-section">
      <input
        v-model="newTodoText"
        @keyup.enter="addTodo"
        placeholder="Add a new task..."
      />
      <button @click="addTodo" class="add-button">Add</button>
      
    </div>

    <div class="todo-sections">
      <div class="pending-section">
        <h2>Pending Tasks</h2>
        <ul class="todo-list">
          <li
            v-for="todo in pendingTodos"
            :key="todo.id"
            draggable="true"
            @dragstart="handleDragStart(todo)"
            @dragover.prevent="handleDragOver(todo)"
            @drop="handleDrop(todo)"
            @dragleave="handleDragLeave(todo)"
            @dragend="handleDragEnd"
            :class="{ 'dragging': draggedTodo === todo, 'drag-over': draggedOverTodo === todo }"
          >
            <input type="checkbox" v-model="todo.completed" />
            <span v-if="!todo.isEditing" @dblclick="editTodo(todo)">{{ todo.text }}</span>
            <input
              v-else
              v-model="todo.editedText"
              @keyup.enter="saveEdit(todo)"
              @blur="saveEdit(todo)"
              class="edit-input"
            />
            <button @click="removeTodo(todo)" class="delete-button">Delete</button>
            <button @click="moveTodoToNextDay(todo)" class="move-button">Move</button>
          </li>
          <p v-if="pendingTodos.length === 0" class="empty-message">No pending tasks.</p>
        </ul>
      </div>

      <div class="completed-section">
        <h2>Completed Tasks</h2>
        <ul class="todo-list">
          <li v-for="todo in completedTodos" :key="todo.id" class="completed">
            <input type="checkbox" v-model="todo.completed" />
            <span v-if="!todo.isEditing" @dblclick="editTodo(todo)">{{ todo.text }}</span>
            <input
              v-else
              v-model="todo.editedText"
              @keyup.enter="saveEdit(todo)"
              @blur="saveEdit(todo)"
              class="edit-input"
            />
          </li>
          <p v-if="completedTodos.length === 0" class="empty-message">No completed tasks.</p>
        </ul>
      </div>

      <p v-if="currentDayTodos.length === 0" class="summary">No tasks for {{ formattedCurrentDate }}! Enjoy your day!</p>
    </div>

    <p v-if="currentDayTodos.length > 0" class="summary">
      {{ pendingTodos.length }} pending, {{ completedTodos.length }} completed for {{ formattedCurrentDate }}
    </p>
  </div>
</template>

<style>
:root {
  --primary-color: #4a90e2; /* A vibrant blue */
  --secondary-color: #50e3c2; /* A mint green */
  --text-color-dark: #333;
  --text-color-light: #666;
  --bg-color-light: #f0f2f5; /* Light grey background */
  --bg-color-card: #ffffff;
  --border-color: #e0e0e0;
  --shadow-light: rgba(0, 0, 0, 0.08);
  --shadow-medium: rgba(0, 0, 0, 0.15);
}

html, body {
  height: 100%; /* Ensure html and body take full height */
  margin: 0;
  padding: 0;
  /* Removed overflow: hidden; */
}

body {
  font-family: 'Segoe UI', 'Roboto', 'Helvetica Neue', Arial, sans-serif;
  background-color: var(--bg-color-light);
  display: flex; /* Use flexbox for centering */
  justify-content: center; /* Center horizontally */
  align-items: flex-start;   /* Align to top */
  min-height: 100vh; /* Ensure body takes at least full viewport height */
  box-sizing: border-box; /* Include padding in element's total width and height */
  padding: 20px; /* Add some padding to prevent content from touching edges */
}

.todo-app {
  background-color: var(--bg-color-card);
  padding: 40px;
  border-radius: 16px;
  box-shadow: 0 10px 30px var(--shadow-medium); /* More pronounced shadow */
  width: 600px;
  max-width: 720px;
  text-align: center;
  animation: fadeInScale 0.6s ease-out forwards; /* New animation */
  /* Removed position: fixed, top, left, transform */
  margin: auto; /* Explicitly center the flex item */
  }

@keyframes fadeInScale {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}

h1 {
  color: var(--primary-color);
  margin-bottom: 30px;
  font-size: 2.8em;
  font-weight: 700;
  letter-spacing: -0.5px;
}

.date-navigation {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
  background-color: #f8f8f8; /* Very light grey */
  padding: 12px 20px;
  border-radius: 10px;
  box-shadow: inset 0 1px 5px var(--shadow-light);
}

.date-navigation h2 {
  font-size: 1.6em; /* Reverted font size */
  margin: 0;
  color: var(--text-color-dark);
  font-weight: 600;
}

.date-input {
  flex-grow: 1;
  margin: 0 10px; /* Space between buttons and input */
  padding: 8px 12px;
  border: 2px solid var(--border-color);
  border-radius: 8px;
  font-size: 20px;
  color: var(--text-color-dark);
  text-align: center;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.date-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(74, 144, 226, 0.2);
}

.nav-button {
  padding: 10px 18px;
  background-color: var(--primary-color);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 20px; /* Reverted font size */
  font-weight: 500;
  transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 5px var(--shadow-light);
}

.nav-button:hover {
  background-color: #3a7bd5;
  transform: translateY(-2px);
}

.nav-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px var(--shadow-light);
}

.input-section {
  display: flex;
  gap: 10px; /* Adjusted gap to match date-navigation */
}

.input-section input {
  flex-grow: 1;
  padding: 14px 18px; /* Reverted padding */
  border: 2px solid var(--border-color);
  border-radius: 10px; /* Reverted border-radius */
  font-size: 18px;
  color: var(--text-color-dark);
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.input-section input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 4px rgba(74, 144, 226, 0.2);
}

.add-button {
  padding: 12px 22px; /* Slightly more padding */
  background: linear-gradient(45deg, var(--secondary-color), #42c8b0); /* Gradient background */
  color: white; /* Changed text color to white for better contrast */
  border: none;
  border-radius: 10px; /* Slightly more rounded corners */
  cursor: pointer;
  font-size: 20px;
  font-weight: 700; /* Bolder font */
  transition: all 0.3s ease; /* Smoother transition for all properties */
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.2); /* More prominent shadow */
}

.add-button:hover {
  background-color: #42c8b0;
  transform: translateY(-2px);
}

.add-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px var(--shadow-light);
}

.todo-sections {
  margin-top: 20px;
  /* Removed max-height and overflow-y from here */
}

.todo-sections h2 {
  font-size: 1.5em;
  color: var(--primary-color);
  margin-bottom: 15px;
  border-bottom: 2px solid var(--primary-color);
  padding-bottom: 5px;
  display: inline-block; /* To make border-bottom only span text */
}

.todo-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.todo-list li {
  display: flex;
  align-items: center;
  padding: 15px 0;
  border-bottom: 1px solid var(--border-color);
  text-align: left;
  transition: background-color 0.3s ease; /* Removed transform */
}

.todo-list li:last-child {
  border-bottom: none;
}

.todo-list li:hover {
  background-color: #fcfcfc;
}

.todo-list li.dragging {
  opacity: 0.5;
  border: 2px dashed var(--primary-color);
  background-color: #e6f0fa;
}

.todo-list li.drag-over {
  border-top: 2px solid var(--primary-color);
}

.todo-list li.drag-over:not(.dragging) {
  transform: translateY(5px);
}

.todo-list li.drag-over:not(.dragging) {
  transform: translateY(5px);
}

.todo-list li input[type="checkbox"] {
  margin-right: 18px;
  transform: scale(1.4); /* Larger checkbox */
  accent-color: var(--secondary-color); /* Green accent */
  cursor: pointer;
  min-width: 20px; /* Ensure checkbox doesn't shrink */
  min-height: 20px;
}

.todo-list li span {
  flex-grow: 1;
  font-size: 1.2em;
  color: var(--text-color-dark);
  word-break: break-word;
  line-height: 1.4;
}

.todo-list li.completed span {
  text-decoration: line-through;
  color: var(--text-color-light);
  font-style: italic;
}

.edit-input {
  flex-grow: 1;
  padding: 5px;
  font-size: 1.2em;
  border: 1px solid var(--border-color);
  border-radius: 5px;
  margin-right: 10px;
}

.delete-button {
  background-color: #ff6b6b; /* Soft red for delete */
  color: white;
  border: none;
  padding: 8px 15px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.1em;
  font-weight: 500;
  margin-left: 20px;
  transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 5px var(--shadow-light);
}

.move-button {
  background-color: #4CAF50; /* Green for move */
  color: white;
  border: none;
  padding: 8px 15px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.1em;
  font-weight: 500;
  margin-left: 10px; /* Smaller margin to be closer to delete button */
  transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 5px var(--shadow-light);
}

.move-button:hover {
  background-color: #45a049;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px var(--shadow-light);
}

.move-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px var(--shadow-light);
}

.delete-button:hover {
  background-color: #e05656;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px var(--shadow-light);
}

.delete-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px var(--shadow-light);
}

.summary {
  margin-top: 30px;
  color: var(--text-color-light);
  font-size: 1.05em;
  font-style: italic;
}

.empty-message {
  color: var(--text-color-light);
  font-style: italic;
  margin-top: 10px;
}

/* Responsive adjustments */
@media (max-width: 650px) {
  html, body {
    overflow: auto; /* Allow scrolling on small screens */
  }

  .todo-app {
    position: static; /* Remove fixed positioning on small screens */
    transform: none;
    padding: 25px;
    border-radius: 0;
    box-shadow: none;
    max-width: 100%;
    height: 100%; /* Take full height on small screens */
    display: flex;
    flex-direction: column;
    justify-content: flex-start; /* Align content to top */
  }

  h1 {
    font-size: 2.2em;
    margin-bottom: 20px;
  }

  .date-navigation {
    flex-wrap: wrap;
    justify-content: center;
    gap: 10px;
    padding: 10px;
  }

  .date-navigation h2 {
    width: 100%;
    text-align: center;
    margin-bottom: 10px;
  }

  .input-section {
    flex-direction: column;
    gap: 10px;
  }

  .add-button {
    width: 100%;
  }

  .todo-list li {
    flex-wrap: wrap;
    justify-content: space-between;
    padding: 10px 0;
  }

  .todo-list li input[type="checkbox"] {
    order: 1;
    margin-right: 10px;
  }

  .todo-list li span {
    order: 2;
    flex-basis: calc(100% - 60px);
    margin-bottom: 5px;
    font-size: 1.1em;
  }

  .delete-button,
  .move-button {
    order: 3;
    margin-left: 0;
    width: 100%;
    margin-top: 10px;
  }

  .summary {
    margin-top: 20px;
    font-size: 0.95em;
  }
}
.export-button {
  padding: 10px 18px;
  background-color: #28a745; /* A nice green */
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 20px;
  font-weight: 600;
  transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 5px var(--shadow-light);
}

.export-button:hover {
  background-color: #218838;
  transform: translateY(-2px);
}

.export-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px var(--shadow-light);
}
</style>
