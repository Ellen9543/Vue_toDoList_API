<script setup>
import { ref, computed, onMounted } from 'vue';


// 跟 setup 一樣會自動執行，vue 生命週期(Lifecycle Hooks) setup => onMounted
onMounted(() => {
  if (token !== '') {
    userLogin()
  }
})

let viewType = ref('LOGIN')
let token = window.localStorage.getItem('token') || '';
const memberInfo = ref({
  nickname: '',
  email: '',
  password: '',
});

const todos = ref([])
const newItem = ref('')

const userRegister = () => {
  fetch('https://todoo.5xcamp.us/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'accept': 'application/json',
    },
    body: JSON.stringify({
      user: {
        nickname: memberInfo.value.nickname,
        email: memberInfo.value.email,
        password: memberInfo.value.password,
      }
    }),
  })
    .then((res) => {
      viewType = 'LOGIN'
      window.location.reload()
      //userLogin()
    });

}

const userLogin = () => {
  fetch('https://todoo.5xcamp.us/users/sign_in', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'authorization': token,
    },
    body: JSON.stringify({
      user: {
        email: memberInfo.value.email,
        password: memberInfo.value.password,
      }
    }),
  })
    .then((res) => {
      const headers = res.headers;
      const authorization = headers.get('Authorization');
      window.localStorage.setItem('token', authorization);
      token = authorization
      return res.json();
    })
    .then((data) => {
      // ES6語法 ?. => data存在的話 才去取 data.nickname
      memberInfo.value.nickname = data?.nickname || '';
      if (memberInfo.value.nickname) {
        getList()
      }
    });
}


const userLogout = () => {
  fetch('https://todoo.5xcamp.us/users/sign_out', {
    method: 'DELETE',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json',
      'authorization': token,
    },
  })
  .then((res) => {
    window.localStorage.clear('token');
    window.location.reload()
  });
}


const getList = () => {
  console.log('getList-token', token)
  fetch('https://todoo.5xcamp.us/todos', {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      'authorization': token,
    },
  })
  .then((res) => {
    console.log('getList', res)
    return res.json();
  })
  .then((data) => {
    todos.value = data.todos
  });
}


const addItem = () => {
  if (newItem.value == '' ) {
    return
  }
  fetch('https://todoo.5xcamp.us/todos', {
    method: 'POST',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json',
      'authorization': token,
    },
    body: JSON.stringify({
      todo: {
        content: newItem.value,
      }
    }),
  })
  .then((res) => {
    newItem.value = ''
    getList()
  });
}

const delItem = (id) => {
  fetch('https://todoo.5xcamp.us/todos/'+id, {
    method: 'DELETE',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json',
      'authorization': token,
    },
    body: JSON.stringify({
      todo: {
        content: newItem.value,
      }
    }),
  })
  .then((res) => {
    getList()
  });
}


const finishItem = (id) => {
  fetch(`https://todoo.5xcamp.us/todos/${id}/toggle`, {
    method: 'PATCH',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json',
      'authorization': token,
    },
  })
  .then((res) => {
    getList()
  });
}

const searchVal = ref('')
const filterList = computed(()=>{
  return todos.value.filter((item)=>{
    return item.content.toLowerCase().includes(searchVal.value.toLowerCase())
  })
})

const changeDisable =(item) => {
  item.isUpdateBtnShow = item.content !== item.originContent;
}


const updateItem = (item) => {
  if (item.content == '') {
    return
  }
  fetch(`https://todoo.5xcamp.us/todos/${item.id}`, {
    method: 'PUT',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json',
      'authorization': token,
    },
    body: JSON.stringify({
      todo: {
        content: item.content,
      }
    }),
  })
  .then((res) => {
    getList()
  });
}
</script>

<template>
  <div class="bg-gray-100 pb-12">
    <!-- Header -->
    <header class="bg-blue-600 text-white p-4">
      <div class="container flex mx-auto justify-between items-center">
        <h1 class="font-bold text-xl">Todo List</h1>
        <div>
          
          <button class="rounded bg-blue-800 py-2 px-4" v-if="token == ''">Login</button>
          <span v-if="token !== ''">{{ memberInfo.nickname }}</span>
          <button class="rounded bg-blue-800 py-2 px-4 mx-4" v-if="token !== ''" @click="userLogout">Logout</button>
        </div>
      </div>
    </header>

    <!-- Main Content -->
    <main class="container mx-auto mt-8">
      <!-- Add Todo Item -->
      <div class="bg-white rounded shadow p-6">
        <h2 class="font-semibold text-lg mb-4">Add New Todo</h2>
        <form id="addTodoForm">
          <input type="text" id="todoInput" class="border w-full p-2" placeholder="Enter new todo item" v-model="newItem">
          <button class="rounded bg-blue-600 mt-2 text-white py-2 px-4" @click.prevent="addItem">Add</button>
        </form>
      </div>

      <!-- Todo List -->
      <div class="bg-white rounded shadow mt-6 p-6">
        <h2 class="font-semibold text-lg mb-4">Todo List</h2>
        <input type="text" v-model="searchVal" class="border w-full p-2 my-2" />
        <ul class="list-disc pl-5">          
          <template v-for="(item, idx) in filterList">
            <li class="flex mb-2 justify-between items-center">
              <label>
                <input type="checkbox" class="mx-2" @click.prevent="finishItem(item.id)" :checked="item.completed_at !== null"/>
                <input type="text" v-model="item.content" @blur="changeDisable(item)" :class="{'line-through text-gray-300': item.completed_at !== null}" />
                <input type="hidden" v-text="item.content" v-once :id="'item-' + item.id"  />
                <!--<span>{{ item.content }}</span>-->
              </label>
              
              <div>
                <button class="rounded bg-red-200 text-white py-1 px-2" v-if="item.isUpdateBtnShow" @click.prevent="updateItem(item)">Update</button>
                <button class="rounded bg-red-600 text-white py-1 px-2 mx-2" @click.prevent="delItem(item.id)">Delete</button>
              </div>
            </li>
          </template>
        </ul>
      </div>
    </main>

    <!-- Modals -->

    <!-- Login Modal -->
    <div
      v-if="token === ''"
      class="flex bg-gray-600 bg-opacity-50 inset-0 fixed items-center justify-center">
      <div class="bg-white rounded shadow p-6">
        <div v-if="viewType == 'LOGIN'">
          <h2 class="font-semibold text-lg mb-4">Login</h2>
          <div>
            <input v-model="memberInfo.email" type="email" class="border mb-4 w-full p-2" placeholder="Email">
            <input v-model="memberInfo.password" type="password" class="border mb-4 w-full p-2" placeholder="Password">
            
            <div class="flex justify-between items-center">
              <button @click="userLogin"
              class="rounded bg-blue-600 text-white py-2 px-4">Login</button>
              <a @click="viewType='REGISTER'" class="text-sky-600 font-semibold underline underline-offset-4 cursor-pointer">註冊</a>
            </div>
          </div>
        </div>
        <div v-else>
          <h2 class="font-semibold text-lg mb-4">Register</h2>
          <div>
            <input v-model="memberInfo.email" type="email" class="border mb-4 w-full p-2" placeholder="Email">
            <input v-model="memberInfo.nickname" type="text" class="border mb-4 w-full p-2" placeholder="NickName">
            <input v-model="memberInfo.password" type="password" class="border mb-4 w-full p-2" placeholder="Password">
            
            <div class="flex justify-between items-center">
              <button @click="userRegister"
              class="rounded bg-blue-600 text-white py-2 px-4">Register</button>
              <a @click="viewType='LOGIN'" class="text-sky-600 font-semibold underline underline-offset-4 cursor-pointer">登入</a>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
