<template>
  <div class="container">
    <h1>ToDo List</h1>
    <div
      class="auth-section"
      v-if="!user.uid"
    >
      <template v-if="!isSignUp">
        <h2>登入</h2>
        <div class="auth-form">
          <input
            type="email"
            placeholder="email"
            v-model="signInField.email"
          />
          <input
            type="password"
            placeholder="password"
            v-model="signInField.password"
          />
          <button @click="signIn">登入</button>
          <p
            v-if="signInField.error"
            class="error"
          >
            {{ signInField.error }}
          </p>
          <p>
            尚未擁有帳號?
            <a
              href="#"
              @click.prevent="toggleAuthMode"
              >註冊</a
            >
          </p>
        </div>
      </template>
      <template v-else>
        <h2>註冊</h2>
        <div class="auth-form">
          <input
            placeholder="email"
            type="email"
            v-model="signUpField.email"
          />
          <input
            type="password"
            placeholder="password"
            v-model="signUpField.password"
          />
          <input
            type="text"
            placeholder="nickname"
            v-model="signUpField.nickname"
          />
          <button
            type="button"
            @click="signUp"
          >
            註冊
          </button>
          <p
            v-if="signUpField.error"
            class="error"
          >
            {{ signUpField.error }}
          </p>
          <!-- <p v-if="signUpRes.uid">UID: {{ signUpRes.uid }}</p> -->
          <p>
            已有帳號?
            <a
              href="#"
              @click.prevent="toggleAuthMode"
              >登入</a
            >
          </p>
        </div>
      </template>
    </div>
    <div
      class="user-info"
      v-if="user.uid"
    >
      <h2>用戶資訊</h2>
      <p>UID: {{ user.uid }}</p>
      <p>暱稱: {{ user.nickname }}</p>
      <button
        type="button"
        @click="signOut"
      >
        登出
      </button>
    </div>
    <div
      class="todo-section"
      v-if="user.uid"
    >
      <h2>我的待辦事項</h2>
      <div class="add-todo">
        <input
          type="text"
          placeholder="新增待辦事項"
          v-model="newTodo"
          @keyup.enter="addTodo(newTodo)"
        />
        <button
          type="button"
          @click="addTodo(newTodo)"
        >
          新增
        </button>
      </div>
      <ul class="todo-list">
        <li
          v-for="todo in todos"
          :key="todo.id"
          :class="{ completed: todo.status }"
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
              確認
            </button>
            <button
              type="button"
              @click="cancelEdit"
            >
              取消
            </button>
          </template>
          <template v-else>
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
              class="edit-button"
            >
              編輯
            </button>
            <button
              type="button"
              @click="deleteTodo(todo)"
            >
              刪除
            </button>
          </template>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import axios from "axios";
import { onMounted, ref } from "vue";
const site = "https://todolist-api.hexschool.io";

// 是否顯示註冊表單
const isSignUp = ref(false);

// 切換登入/註冊模式
const toggleAuthMode = () => {
  isSignUp.value = !isSignUp.value;
};

// 初始化各種 ref
const signUpField = ref({
  email: "",
  password: "",
  nickname: "",
  error: "",
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

// 設定 token
const setToken = (token, exp) => {
  const expirationDate = new Date(exp * 1000);
  document.cookie = `hexToken=${token}; expires=${expirationDate.toUTCString()}; path=/`;
};

// 清除 token
const clearToken = () => {
  document.cookie = `hexToken=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;`;
};

// 註冊功能
const signUp = async () => {
  try {
    const res = await axios.post(`${site}/users/sign_up`, signUpField.value);
    signUpRes.value = res.data;

    // 註冊後直接登入
    signInField.value = {
      email: signUpField.value.email,
      password: signUpField.value.password,
    };
    signIn();
    signUpField.value = { email: "", password: "", nickname: "" };
  } catch (error) {
    console.error(error.response.data.message);
    signUpField.value.error = error.response.data.message;
  }
};

// 登入功能
const signIn = async () => {
  try {
    const res = await axios.post(`${site}/users/sign_in`, signInField.value);
    signInRes.value = res.data;
    setToken(res.data.token, res.data.exp);
    signInField.value = { email: "", password: "", error: "" };
    checkSignIn();
    getTodos();
  } catch (error) {
    console.error(error.response.data.message);
    signInField.value.error = error.response.data.message;
  }
};

// 登出功能
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

// 新增待辦事項
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
      getTodos();
      newTodo.value = "";
    }
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 獲取待辦事項
const getTodos = async () => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.get(`${site}/todos`, {
      headers: { Authorization: token },
    });
    todos.value = res.data.data;
  } catch (error) {
    console.error(error.response);
  }
};

// 編輯待辦事項
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
    getTodos();
    editingTodo.value = {};
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 刪除待辦事項
const deleteTodo = async (todo) => {
  const token = getToken();
  if (!token) return;

  try {
    const res = await axios.delete(`${site}/todos/${todo.id}`, {
      headers: { Authorization: token },
    });
    if (res.status === 200) {
      getTodos();
    }
  } catch (error) {
    console.error(error.response.data.message);
  }
};

// 更新待辦事項狀態
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
      getTodos();
    }
  } catch (error) {
    console.error("Error updating status:", error.response.data.message);
  }
};
</script>

<style scoped>
/* 容器樣式 */
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f7f7f7;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

/* 標題樣式 */
h1 {
  text-align: center;
  margin-bottom: 20px;
}

/* 表單樣式 */
.auth-section,
.todo-section {
  margin-bottom: 20px;
}

.auth-form,
.add-todo {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

input[type="email"],
input[type="password"],
input[type="text"] {
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 16px;
}

button {
  padding: 10px;
  background-color: #5b99c2;
  border: none;
  border-radius: 4px;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

button:hover {
  background-color: #86ab89;
}

.error {
  color: red;
}

/* 待辦事項樣式 */
.todo-list {
  list-style-type: none;
  padding: 0;
}

.todo-list li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-bottom: 10px;
  background-color: white;
}

.todo-list li.completed label {
  text-decoration: line-through;
  color: #777;
}

.todo-list input[type="checkbox"] {
  margin-right: 10px;
}

.todo-list label {
  flex-grow: 1;
}

.todo-list button {
  margin-left: 10px;
  background-color: #dc3545;
}

.todo-list button:hover {
  background-color: #c82333;
}

.todo-list .edit-button {
  background-color: #86ab89;
}

.todo-list .edit-button:hover {
  background-color: #697565;
}

.user-info {
  padding: 10px;
  margin: 10px 0;
  background-color: #fffbe6;
  border: 2px solid #fffbe6;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}
</style>
