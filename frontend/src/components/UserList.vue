<template>
  <div>
    <!-- Header / Controls Panel -->
    <div class="panel">
      <div class="panel-inner">
        <div class="stack-wrap">
          <h2 class="title"><i class="bi bi-people-fill" style="color: var(--primary-2);"></i> รายชื่อทั้งหมด</h2>
          <div class="spacer"></div>
          <button class="btn btn-primary" @click="openAddUserModal"><i class="bi bi-plus-lg"></i> เพิ่มผู้ใช้</button>
        </div>

        <div class="stack-wrap mt-3">
          <input v-model="searchQuery" type="text" class="input" style="flex:1 1 320px" placeholder="🔍 ค้นหา (ชื่อ, นามสกุล, อายุ, ความสนใจ, รายละเอียด)...">

          <select v-model="selectedGender" class="input select" style="flex:1 1 200px">
            <option value="">-- กรองเพศ --</option>
            <option value="ชาย">ชาย</option>
            <option value="หญิง">หญิง</option>
            <option value="ไม่ระบุ">ไม่ระบุ</option>
          </select>

          <select v-model="sortKey" class="input select" style="flex:1 1 220px">
            <option value="">-- เรียงลำดับ --</option>
            <option value="firstname">ชื่อ (A-Z)</option>
            <option value="lastname">นามสกุล (A-Z)</option>
            <option value="age">อายุ (น้อย → มาก)</option>
          </select>
        </div>
      </div>
    </div>

    <!-- Data Table -->
    <div class="mt-4">
      <div class="grid-table cols-users grid-header">
        <div class="cell">ชื่อ</div>
        <div class="cell">นามสกุล</div>
        <div class="cell">อายุ</div>
        <div class="cell">เพศ</div>
        <div class="cell">ความสนใจ</div>
        <div class="cell">รายละเอียด</div>
        <div class="cell" style="text-align:right">จัดการ</div>
      </div>

      <transition-group name="list" tag="div">
        <div class="grid-table cols-users grid-row" v-for="user in sortedAndFilteredUsers" :key="user.id">
          <div class="cell" style="font-weight:600">{{ user.firstname }}</div>
          <div class="cell">{{ user.lastname }}</div>
          <div class="cell">{{ user.age }}</div>
          <div class="cell">
            <span class="badge" :class="{
              male: user.gender === 'ชาย',
              female: user.gender === 'หญิง',
              na: user.gender !== 'ชาย' && user.gender !== 'หญิง'
            }">{{ user.gender }}</span>
          </div>
          <div class="cell wrap">{{ user.interests }}</div>
          <div class="cell wrap" style="color:#cbd5e1">{{ user.description }}</div>
          <div class="cell actions">
            <button class="btn btn-outline" @click="editUser(user)"><i class="bi bi-pencil-square"></i> แก้ไข</button>
            <button class="btn btn-danger" @click="deleteUser(user.id)"><i class="bi bi-trash"></i> ลบ</button>
          </div>
        </div>
      </transition-group>
    </div>

    <!-- Modal -->
    <transition name="fade">
      <div v-if="showAddUserModal" class="modal-backdrop">
        <div class="panel modal-card">
          <div class="modal-header">
            <div style="font-weight:700"><i class="bi bi-person-plus" style="color: var(--primary-2);"></i> {{ editingUser ? 'แก้ไขข้อมูล' : 'เพิ่มผู้ใช้' }}</div>
            <button class="btn btn-outline" @click="closeModal"><i class="bi bi-x-lg"></i></button>
          </div>
          <div class="modal-body">
            <div class="stack-wrap">
              <input v-model="newUser.firstname" type="text" class="input" style="flex:1 1 180px" placeholder="ชื่อ">
              <input v-model="newUser.lastname" type="text" class="input" style="flex:1 1 180px" placeholder="นามสกุล">
              <input v-model="newUser.age" type="number" min="0" class="input" style="width:120px" placeholder="อายุ">
            </div>
            <div class="stack-wrap mt-3">
              <select v-model="newUser.gender" class="input select" style="flex:1 1 180px">
                <option value="หญิง">หญิง</option>
                <option value="ชาย">ชาย</option>
                <option value="ไม่ระบุ">ไม่ระบุ</option>
              </select>
              <input v-model="newUser.interests" type="text" class="input" style="flex:2 1 260px" placeholder="ความสนใจ">
            </div>
            <div class="mt-3">
              <textarea v-model="newUser.description" rows="3" class="input" style="width:100%" placeholder="รายละเอียด"></textarea>
            </div>
          </div>
          <div class="modal-footer">
            <div class="spacer"></div>
            <button class="btn btn-outline" @click="closeModal">ยกเลิก</button>
            <button class="btn btn-primary" @click="editingUser ? updateUser() : addUser()">{{ editingUser ? 'อัปเดต' : 'เพิ่ม' }}</button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      users: [],
      searchQuery: "",
      selectedGender: "",
      sortKey: "",
      sortDirection: "asc",
      showAddUserModal: false,
      editingUser: null,
      newUser: { firstname: "", lastname: "", age: "", gender: "หญิง", interests: "", description: "" },
    };
  },
  computed: {
    sortedAndFilteredUsers() {
      let filtered = this.users;

      // ✅ กรองตามเพศ
      if (this.selectedGender) {
        filtered = filtered.filter(user => user.gender === this.selectedGender);
      }

      // ✅ ค้นหาจากชื่อ / นามสกุล / อายุ / ความสนใจ / รายละเอียด (ไม่สนตัวพิมพ์เล็ก-ใหญ่)
      const q = this.searchQuery.trim().toLowerCase();
      if (q) {
        filtered = filtered.filter(user => {
          const firstname = (user.firstname ?? '').toString().toLowerCase();
          const lastname = (user.lastname ?? '').toString().toLowerCase();
          const age = (user.age ?? '').toString().toLowerCase();
          const interests = (user.interests ?? '').toString().toLowerCase();
          const description = (user.description ?? '').toString().toLowerCase();
          return (
            firstname.includes(q) ||
            lastname.includes(q) ||
            age.includes(q) ||
            interests.includes(q) ||
            description.includes(q)
          );
        });
      }

      // ✅ เรียงลำดับ
      if (this.sortKey) {
        filtered = [...filtered].sort((a, b) => {
          let result = 0;
          if (a[this.sortKey] < b[this.sortKey]) result = -1;
          if (a[this.sortKey] > b[this.sortKey]) result = 1;
          return this.sortDirection === "asc" ? result : -result;
        });
      }

      return filtered;
    }
  },
  mounted() {
    this.fetchUsers();
  },
  methods: {
    async fetchUsers() {
      try {
        const response = await axios.get("http://localhost:8000/users");
        this.users = response.data;
      } catch (error) {
        console.error("Error fetching users:", error);
      }
    },
    openAddUserModal() {
      this.newUser = { firstname: "", lastname: "", age: "", gender: "หญิง", interests: "", description: "" };
      this.editingUser = null;
      this.showAddUserModal = true;
    },
    async addUser() {
      try {
        await axios.post("http://localhost:8000/users", this.newUser);
        this.fetchUsers();
        this.closeModal();
      } catch (error) {
        console.error("Error adding user:", error);
      }
    },
    async deleteUser(id) {
      if (confirm("คุณต้องการลบผู้ใช้นี้หรือไม่?")) {
        try {
          await axios.delete(`http://localhost:8000/users/${id}`);
          this.fetchUsers();
        } catch (error) {
          console.error("Error deleting user:", error);
        }
      }
    },
    editUser(user) {
      this.newUser = { ...user };
      this.editingUser = user.id;
      this.showAddUserModal = true;
    },
    async updateUser() {
      try {
        await axios.put(`http://localhost:8000/users/${this.editingUser}`, this.newUser);
        this.fetchUsers();
        this.closeModal();
      } catch (error) {
        console.error("Error updating user:", error);
      }
    },
    closeModal() {
      this.showAddUserModal = false;
      this.newUser = { firstname: "", lastname: "", age: "", gender: "หญิง", interests: "", description: "" };
      this.editingUser = null;
    }
  }
};
</script>
