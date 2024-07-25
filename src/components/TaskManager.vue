<template>
  <div>
    <h1>Task Manager</h1>
    <button @click="executeTasks">Начать процесс</button>
    <ul>
      <li v-for="task in tasks" :key="task.id">
        <Task
          :task="task"
          :status="taskStatus[task.id]"
          @cancel="cancelTask(task.id)"
        />
      </li>
    </ul>
  </div>
</template>

<script>
import Task from "./Task.vue";

export default {
  components: {
    Task,
  },
  data() {
    return {
      maxConcurrentTasks: 2, // Максимальное количество одновременно выполняемых задач
      tasks: [], // Список всех задач
      taskStatus: {}, // Состояния всех задач
      taskTimeouts: {}, // Таймауты для задач
    };
  },
  methods: {
    // Метод для добавления новой задачи
    addTask(task, priority, dependencies = [], timeout = null) {
      const id = `task${this.tasks.length + 1}`;
      this.tasks.push({ id, task, priority, dependencies }); // Добавление задачи в список
      this.taskStatus[id] = "pending"; // Установка начального статуса задачи
      this.taskTimeouts[id] = timeout;
    },
    // Метод для выполнения всех задач
    async executeTasks() {
      // Сортировка задач по приоритету (от большего к меньшему)
      const sortedTasks = this.tasks.sort((a, b) => b.priority - a.priority);
      const executingTasks = new Set(); // отслеживание задач

      // Функция для выполнения одной задачи
      const executeTask = async (task) => {
        if (this.taskStatus[task.id] === "cancelled") {
          console.log(`Задача ${task.id} отменена и не будет выполняться`);
          return;
        }

        this.taskStatus[task.id] = "running"; // Установка статуса "running" для задачи
        console.log(`Запуск задачи ${task.id}`);
        const timeout = this.taskTimeouts[task.id];
        let timeoutId;
        try {
          if (timeout) {
            // Обработка таймаута для задачи
            const timeoutPromise = new Promise((_, reject) => {
              timeoutId = setTimeout(
                () => reject(new Error("Timeout")),
                timeout
              );
            });
            await Promise.race([task.task(), timeoutPromise]);
          } else {
            await task.task();
          }

          if (this.taskStatus[task.id] !== "cancelled") {
            this.taskStatus[task.id] = "completed"; // Установка статуса "completed" для задачи
            console.log(`Задача ${task.id} завершена`);
            checkAndProcessWaitingTasks(); // Проверяем и запускаем ожидающие задачи
          }
        } catch (error) {
          if (this.taskStatus[task.id] !== "cancelled") {
            this.taskStatus[task.id] = "failed"; // Установка статуса "failed" для задачи
            console.error(`Ошибка при выполнении задачи ${task.id}:`, error);
          }
        } finally {
          clearTimeout(timeoutId);
        }
      };

      const processTask = async (task) => {
        if (this.taskStatus[task.id] === "cancelled") {
          console.log(`Задача ${task.id} отменена и пропускается`);
          return;
        }
        // Проверка, завершены ли все зависимости задачи
        if (
          task.dependencies.every((dep) => this.taskStatus[dep] === "completed")
        ) {
          if (this.taskStatus[task.id] !== "running") {
            // Запуск задачи, если еще не выполняется
            if (executingTasks.size < this.maxConcurrentTasks) {
              executingTasks.add(
                executeTask(task).finally(() => executingTasks.delete(task))
              );
            } else {
              await Promise.race(executingTasks);
              executingTasks.add(
                executeTask(task).finally(() => executingTasks.delete(task))
              );
            }
          }
        } else {
          this.taskStatus[task.id] = "waiting"; // Установка статуса "waiting" для задачи
          console.log(
            `Задача ${task.id} ожидает завершения зависимостей: ${task.dependencies}`
          );
        }
      };
      // Функция для проверки и запуска ожидающих задач
      const checkAndProcessWaitingTasks = () => {
        for (const task of this.tasks) {
          if (
            this.taskStatus[task.id] === "waiting" &&
            task.dependencies.every(
              (dep) => this.taskStatus[dep] === "completed"
            )
          ) {
            processTask(task);
          }
        }
      };
      // Запуск всех задач
      for (const task of sortedTasks) {
        await processTask(task);
      }

      await Promise.all(executingTasks);
      checkAndProcessWaitingTasks();
    },
    // Метод для получения статусов задач
    getStatus() {
      return this.taskStatus;
    },
    // Метод для отмены задачи по ID
    cancelTask(taskId) {
      const task = this.tasks.find((t) => t.id === taskId);
      if (task) {
        // Задача найдена вызываем метод
        this._cancelTaskRecursively(task);
      }
    },
    // Вспомогательный метод для рекурсивной отмены задач и их зависимостей
    _cancelTaskRecursively(task) {
      if (
        this.taskStatus[task.id] !== "completed" &&
        this.taskStatus[task.id] !== "cancelled"
      ) {
        this.taskStatus[task.id] = "cancelled";
        const dependentTasks = this.tasks.filter(
          (
            t // смотрим на зависимости задачи есть ли
          ) => t.dependencies.includes(task.id)
        );
        for (const depTask of dependentTasks) {
          // вызывывем себя чтобы отмнеить зависимые задачи
          this._cancelTaskRecursively(depTask);
        }
      }
    },
  },
  mounted() {
    this.addTask(async () => {
      console.log("Выполнение задачи 1");
      await new Promise((resolve) => setTimeout(resolve, 2000));
      console.log("Задача 1 завершена");
    }, 2);

    this.addTask(
      async () => {
        console.log("Выполнение задачи 2");
        await new Promise((resolve) => setTimeout(resolve, 1000));
        console.log("Задача 2 завершена");
      },
      1,
      ["task1"],
      1500
    );

    this.addTask(async () => {
      console.log("Выполнение задачи 3");
      await new Promise((resolve) => setTimeout(resolve, 500));
      console.log("Задача 3 завершена");
    }, 3);

    this.addTask(
      async () => {
        console.log("Выполнение задачи 4");
        await new Promise((resolve) => setTimeout(resolve, 3000));
        console.log("Задача 4 завершена");
      },
      4,
      ["task1", "task2"],
      3500
    );
    this.addTask(
      async () => {
        console.log("Выполнение задачи 5");
        await new Promise((resolve) => setTimeout(resolve, 2500));
        console.log("Задача 5 завершена");
      },
      5,
      ["task2"],
      3000
    );
  },
};
</script>
