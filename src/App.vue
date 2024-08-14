<template>
  <main class="flex min-h-[600px] w-[600px] flex-col rounded-3xl border p-6 shadow-lg">
    <div>
      <form class="flex" @submit.prevent="handleSubmit">
        <input
          class="mr-3 min-h-10 w-full rounded-full border px-5 py-2 focus:outline-none"
          v-model="addTodoText"
          type="text"
          placeholder="Add todo"
        />
        <button type="submit" class="rounded-full border bg-gray-500 px-6 text-white">Add</button>
      </form>

      <div class="flex justify-end pt-3">
        <TodoFilterDropdown
          :options="filterOptions"
          :selectedFilter="selectedFilter"
          @change="handleFilterChange"
        />
      </div>
    </div>

    <ul class="flex-grow overflow-y-auto pt-3" data-testid="todo_ul" ref="todoListRef">
      <li
        v-for="(todo, index) in filteredTodos"
        :key="todo.id"
        class="flex items-center justify-between rounded-full px-5 py-3"
        :class="{ 'bg-gray-100': index % 2 === 0 }"
      >
        <div class="flex-grow">
          <div v-if="todo.id === selectedTodo?.id" class="flex flex-grow">
            <input
              v-model="selectedTodo.completed"
              type="checkbox"
              class="custom-checkbox mr-3 h-6 w-6 accent-cyan-500"
            />
            <input
              v-model="selectedTodo.title"
              class="w-full flex-grow rounded-full px-3 py-0.5 outline outline-2 outline-gray-300 mr-2"
            />
            <div class="flex">
              <button
                @click="selectedTodo = null"
                class="mr-2 rounded-full border bg-gray-500 px-6 py-0.5 text-white"
              >
                Cancel
              </button>
              <button
                @click="handleUpdateTodo"
                class="rounded-full border bg-cyan-500 px-6 py-0.5 text-white"
              >
                Save
              </button>
            </div>
          </div>

          <div v-else class="flex justify-between">
            <div class="flex">
              <div class="mr-3 fill-current from-current">
                <IconDone class="text-gray-300" v-if="todo.completed" />
                <IconPending v-else />
              </div>
              <div>{{ todo.title }}</div>
            </div>

            <div>
              <button
                @click="handleSelectTodo(todo)"
                class="mr-2 rounded-full border bg-gray-500 px-6 py-0.5 text-white"
              >
                Edit
              </button>
              <button
                @click="handleDeleteTodo(todo.id)"
                class="rounded-full border bg-red-400 px-6 py-0.5 text-white"
              >
                Del
              </button>
            </div>
          </div>
        </div>
      </li>
    </ul>
  </main>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue';
import { klona } from 'klona/json';
import Sortable from 'sortablejs';
import IconDone from '@/components/icons/IconDone.vue';
import IconPending from '@/components/icons/IconPending.vue';
import TodoFilterDropdown from './TodoFilterDropdown.vue';

import {
  getTodos,
  addTodo,
  deleteTodo,
  updateTodo,
  updateTodoSortIndex,
  type Todo
} from '@/api/todo';

const todos = ref<Todo[]>([]);
const selectedTodo = ref<Todo | null>(null);
const addTodoText = ref('');
const todoListRef = ref<HTMLUListElement>();
const sortable = ref<Sortable>();
const filterOptions = ['All', 'Progress', 'Completed'];
const selectedFilter = ref(filterOptions[0]);
const filteredTodos = computed(() => {
  switch (selectedFilter.value) {
    case 'All':
      return todos.value;
    case 'Progress':
      return todos.value.filter((todo) => todo.completed);
    case 'Completed':
      return todos.value.filter((todo) => !todo.completed);
    default:
      return [];
  }
});

watch(selectedFilter, (newFilter) => {
  if (sortable.value) {
    sortable.value.option('disabled', newFilter !== 'All');
  }
});

onMounted(() => {
  fetchTodos();
  if (todoListRef.value) {
    sortable.value = new Sortable(todoListRef.value, { onEnd: handleOnEnd });
  }
});

onBeforeUnmount(() => {
  sortable.value?.destroy();
});

async function fetchTodos() {
  try {
    const res = await getTodos();
    todos.value = res.data;
  } catch (err) {
    console.error('getTodos err', err);
  }
}

async function handleSubmit() {
  if (!addTodoText.value.trim()) return;
  try {
    await addTodo({ title: addTodoText.value, completed: false });
    addTodoText.value = '';
  } catch (err) {
    console.error('handleSubmit err', err);
  }

  fetchTodos();
}

async function handleDeleteTodo(id: string) {
  try {
    await deleteTodo(id);
  } catch (err) {
    console.error('handleDeleteTodo err', err);
  }

  fetchTodos();
}

async function handleUpdateTodo() {
  if (!selectedTodo.value) return;
  const { completed, title, id } = selectedTodo.value;
  try {
    await updateTodo(id, {
      title: title,
      completed: completed
    });
    selectedTodo.value = null;
  } catch (error) {
    console.error('updateTodo err', error);
  }

  fetchTodos();
}

async function handleSortTodo(todos: Todo[]) {
  try {
    await updateTodoSortIndex(todos);
  } catch (error) {
    console.error('handleSortTodo err', error);
  }
}

function handleSelectTodo(todo: Todo) {
  selectedTodo.value = klona(todo);
}

function handleFilterChange(option: string) {
  selectedFilter.value = option;
}

function handleOnEnd(event: Sortable.SortableEvent) {
  const { oldIndex, newIndex } = event;
  if (oldIndex !== undefined && newIndex !== undefined) {
    let updatedTodos = [...todos.value];
    const [movedTodo] = updatedTodos.splice(oldIndex, 1);
    updatedTodos.splice(newIndex, 0, movedTodo);

    updatedTodos = updateSortIndex(updatedTodos);
    todos.value = updatedTodos;
    handleSortTodo(updatedTodos);
  }
}

function updateSortIndex(todos: Todo[]) {
  return todos.map((todo, index) => ({
    ...todo,
    sortIndex: index
  }));
}
</script>

<style scoped></style>
