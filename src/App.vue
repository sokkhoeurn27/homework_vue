<template>
  <div class="min-h-screen bg-slate-950 text-slate-100 p-6 sm:p-10">
    <div class="mx-auto max-w-3xl rounded-3xl border border-slate-700 bg-slate-900/90 p-8 shadow-2xl shadow-slate-950/40">
      <header class="mb-8">
        <h1 class="text-4xl font-semibold tracking-tight">Task Manager</h1>
        <p class="mt-2 text-slate-400">Add tasks with priority, mark them done, and see your progress.</p>
      </header>

      <section class="mb-8 space-y-4">
        <div class="grid gap-3 sm:grid-cols-[1fr_auto]">
          <div class="grid gap-3 sm:grid-cols-[1fr_160px]">
            <input
              v-model="newTask"
              @keyup.enter="addTask"
              type="text"
              placeholder="Enter a new task"
              class="w-full rounded-2xl border border-slate-700 bg-slate-950 px-4 py-3 text-slate-100 outline-none transition focus:border-slate-500 focus:ring-2 focus:ring-slate-500/40"
            />
            <select
              v-model="newPriority"
              class="rounded-2xl border border-slate-700 bg-slate-950 px-4 py-3 text-slate-100 outline-none transition focus:border-slate-500 focus:ring-2 focus:ring-slate-500/40"
            >
              <option value="Low">Low</option>
              <option value="Medium">Medium</option>
              <option value="High">High</option>
            </select>
          </div>
          <button
            type="button"
            @click="addTask"
            class="rounded-2xl bg-emerald-500 px-6 py-3 text-sm font-semibold text-slate-950 transition hover:bg-emerald-400"
          >
            Add
          </button>
        </div>

        <div class="rounded-3xl border border-slate-700 bg-slate-950/60 p-5 text-slate-300">
          <p class="text-sm">{{ doneCount }} of {{ tasks.length }} done</p>
        </div>
      </section>

      <section>
        <ul class="space-y-3">
          <li
            v-for="task in tasks"
            :key="task.id"
            class="flex items-center justify-between rounded-3xl border border-slate-700 bg-slate-950/80 p-4"
          >
            <div class="flex items-center gap-4">
              <label class="flex items-center gap-3">
                <input
                  type="checkbox"
                  v-model="task.done"
                  class="h-5 w-5 rounded border-slate-600 bg-slate-900 text-emerald-500 focus:ring-emerald-500"
                />
                <span :class="{ 'line-through text-slate-500': task.done }" class="text-slate-100">{{ task.title }}</span>
              </label>
            </div>

            <div class="flex items-center gap-3">
              <span
                class="rounded-full px-3 py-1 text-xs font-semibold uppercase tracking-wide"
                :class="priorityClasses(task.priority)"
              >
                {{ task.priority }}
              </span>
              <button
                @click="removeTask(task.id)"
                class="rounded-2xl border border-slate-700 bg-slate-900 px-3 py-2 text-slate-300 transition hover:border-rose-400 hover:text-rose-300"
              >
                Remove
              </button>
            </div>
          </li>
        </ul>
      </section>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const tasks = ref([
  { id: 1, title: 'Buy groceries', priority: 'Medium', done: false },
  { id: 2, title: 'Read Vue guide', priority: 'High', done: true },
])

const newTask = ref('')
const newPriority = ref('Medium')
let nextId = 3

const doneCount = computed(() => tasks.value.filter((task) => task.done).length)

function addTask() {
  const title = newTask.value.trim()
  if (!title) {
    return
  }

  tasks.value.push({
    id: nextId++,
    title,
    priority: newPriority.value,
    done: false,
  })

  newTask.value = ''
  newPriority.value = 'Medium'
}

function removeTask(id) {
  tasks.value = tasks.value.filter((task) => task.id !== id)
}

function priorityClasses(priority) {
  return {
    'bg-emerald-500 text-slate-950': priority === 'Low',
    'bg-amber-500 text-slate-950': priority === 'Medium',
    'bg-rose-500 text-slate-950': priority === 'High',
  }
}
</script>
