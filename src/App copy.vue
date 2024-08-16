<script setup>
import axios from "axios";
import { onMounted, ref, watch } from "vue";
const site = "https://todolist-api.hexschool.io";

const signUpfield = ref({
  email: "",
  password: "",
  nickname: "",
});

const signInField = ref({
  email: "",
  password: "",
  error: "",
});

const signUpRes = ref("");
const signInRes = ref("");

const user = ref({
  uid: "",
  nickname: "",
});

const newTodo = ref("");
const todos = ref([]);
const editingTodo = ref({});

// 取得 token
const getToken = () => {
  return document.cookie.replace(
    /(?:(?:^|.*;\s*)hexToken\s*=\s*([^;]*).*$)|^.*$/,
    "$1"
  );
};

// 设定 token
const setToken = (token, exp) => {
  const expirationDate = new Date(exp * 1000);
  document.cookie = `hexToken=${token}; expires=${expirationDate.toUTCString()}; path=/`;
};

// 清除 token
const clearToken = () => {
  document.cookie = `hexToken=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;`;
};

const signUp = async () => {
  try {
    const res = await axios.post(`${site}/users/sign_up`, signUpfield.value);
    signUpRes.value = res.data;
    // 清空form
    signUpfield.value = { email: "", password: "", nickname: "" };
  } catch (error) {
    console.error(error.response.data.message);
  }
};

const signIn = async () => {
  try {
    const res = await axios.post(`${site}/users/sign_in`, signInField.value);
    signInRes.value = res.data;

    setToken(res.data.token, res.data.exp);
    console.log("Token set:", document.cookie);

    // 清空form
    signInField.value = { email: "", password: "", error: "" };

    console.log("成功登入");

    checkSignIn();
    getTodos();
  } catch (error) {
    console.error(error.response.data.message);
    signInField.value.error = error.response.data.message;
  }
};

// 登出
const signOut = async () => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.post(
      `${site}/users/sign_out`,
      {},
      {
        headers: { Authorization: token },
      }
    );
    if (res.status === 200) {
      clearToken();
      user.value = { uid: "", nickname: "" };
      todos.value = [];
      console.log("Cookie cleared:", document.cookie);
    }
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 驗證是否登入
const checkSignIn = async () => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.get(`${site}/users/checkout`, {
      headers: { Authorization: token },
    });
    user.value = { uid: res.data.uid, nickname: res.data.nickname };
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 页面加载时验证登入状态并获取 todos
onMounted(() => {
  checkSignIn();
  getTodos();
});

// 添加 todo
const addTodo = async (todo) => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.post(
      `${site}/todos`,
      { content: todo },
      {
        headers: { Authorization: token },
      }
    );
    if (res.status === 201) {
      console.log("成功新增 todo");
      getTodos();
      newTodo.value = ""; // 清空输入框
    }
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 获取 todos
const getTodos = async () => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.get(`${site}/todos`, {
      headers: { Authorization: token },
    });
    todos.value = res.data.data;
    console.log("todos", todos.value);
    console.log("已从 API 成功获取 todos");
  } catch (error) {
    console.error(error.response);
  }
};

// 编辑 todo
const editTodo = (todo) => {
  editingTodo.value = { ...todo };
};

const cancelEdit = () => {
  editingTodo.value = {};
};

const confirmEdit = async (todo) => {
  const token = getToken();
  if (!token) return;

  try {
    await axios.put(
      `${site}/todos/${todo.id}`,
      { content: editingTodo.value.content },
      {
        headers: { Authorization: token },
      }
    );
    console.log("成功更新 todo content");
    getTodos(); // 更新列表
    editingTodo.value = {};
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 删除 todo
const deleteTodo = async (todo) => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.delete(`${site}/todos/${todo.id}`, {
      headers: { Authorization: token },
    });
    if (res.status === 200) {
      console.log("成功删除 todo");
      getTodos(); // 更新列表
    }
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 更新 todo 状态
const updateStatus = async (todo) => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.patch(
      `${site}/todos/${todo.id}/toggle`,
      { id: todo.id },
      {
        headers: { Authorization: token },
      }
    );
    if (res.status === 200) {
      console.log("成功更新 todo status");
      getTodos(); // 更新列表
    }
  } catch (error) {
    console.error("Error updating status:", error.response.data.message);
  }
};
</script>

<template>
  <div>
    <h2>註冊</h2>
    <div>
      <input
        placeholder="email"
        type="email"
        v-model="signUpfield.email"
      />
      <input
        type="text"
        placeholder="password"
        v-model="signUpfield.password"
      />
      <input
        type="text"
        placeholder="nickname"
        v-model="signUpfield.nickname"
      />
      <br />
      <button
        type="button"
        @click="signUp"
      >
        Sign Up
      </button>
      <p>UID:{{ signUpRes.uid }}</p>
    </div>
    <hr />
    <h2>登入</h2>
    <div>
      <input
        type="email"
        placeholder="email"
        v-model="signInField.email"
      />
      <input
        type="text"
        placeholder="password"
        v-model="signInField.password"
      />
      <br />
      <button @click="signIn">Sign in</button>
      <p>token:{{ signInRes.token }}</p>
      <p>error:{{ signInField.error }}</p>
    </div>
    <hr />
    <h2>驗證登入狀態</h2>
    <div v-if="user.uid">
      <p>UID:{{ user.uid }}</p>
      <p>Nickname:{{ user.nickname }}</p>
    </div>
    <div v-else>
      <p>請先登入</p>
    </div>
    <hr />
    <h2>登出</h2>
    <div v-if="user.uid">
      <p>UID:{{ user.uid }}</p>
      <button
        type="button"
        @click="signOut"
      >
        Sign out
      </button>
    </div>
    <div v-else>
      <p>請先登入</p>
    </div>
    <hr />
    <h2>Todo List</h2>
    <p>todos:{{ todos }}</p>

    <div v-if="user.uid">
      <input
        type="text"
        placeholder="enter some task"
        v-model="newTodo"
        @keyup.enter="addTodo(newTodo)"
      />
      <button
        type="button"
        @click="addTodo(newTodo)"
      >
        Add
      </button>
      <hr />
      <div v-if="todos">
        editingTodo:{{ editingTodo }}
        <ul>
          <li
            v-for="todo in todos"
            :key="todo.id"
          >
            <template v-if="editingTodo.id === todo.id">
              <input
                type="text"
                v-model="editingTodo.content"
              />
              <button
                type="button"
                @click="confirmEdit(todo)"
              >
                confirm
              </button>
              <button
                type="button"
                @click="cancelEdit"
              >
                cancel
              </button>
            </template>
            <template v-else>
              {{ todo.status }}
              <input
                type="checkbox"
                name="todo"
                v-model="todo.status"
                @change="updateStatus(todo)"
              />
              <label for="todo">{{ todo.content }}</label>
              <button
                type="button"
                @click="editTodo(todo)"
              >
                edit
              </button>
              <span @click="deleteTodo(todo)">X</span>
            </template>
          </li>
        </ul>
      </div>
    </div>
    <div v-else>
      <p>請先登入</p>
    </div>
  </div>
</template>

<style scoped>
ul {
  list-style-type: none;
  padding: 0;
}

li {
  margin: 10px 0;
}
</style>
