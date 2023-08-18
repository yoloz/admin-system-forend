<script lang="ts" setup>
import { ref, reactive } from 'vue'
import {
    Search,
    MoreFilled,
    Upload,
    Download
} from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { downloadBlobFile } from '@ued/utils/src/utils'
import dayjs from 'dayjs'
import { storeToRefs } from 'pinia'

import { useUserStore } from '@/stores/userStore'

import UserDrawer from './components/user-drawer.vue'
import CreateUserDialog from './components/create-user-dialog.vue'
import {
    queryUserList,
    removeUser,
    exportData,
    importData
} from './services/userServices'
import { IUserForm, IUserList } from './entity/user'

const { getLoginUser } = storeToRefs(useUserStore())

const createUserDialogRef = ref()

const tableListRef = ref()
const searchUser = ref()
const uploadRef = ref()

const loading = ref<boolean>(false)
const file = ref()
const userDrawerRef = ref()
// const btnLoading = ref<boolean>(false)
// 列表选择
const checked = ref<Array<number>>([])
const params = reactive<IUserList.RequestForm>({
    pageNumber: 1,
    pageSize: 15,
    totalRow: -1
})

// 添加用户对话框
const addUser = () => {
    createUserDialogRef.value.open('add')
}

// 批量删除
const onBatchDeletion = () => {
    const ids = checked.value
    ElMessageBox.confirm('确认删除所选用户吗?', '删除', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
    }).then(() => {
        removeUser(ids).then((res) => {
            ElMessage({ type: 'success', message: '删除成功' })
            refreshTable()
        })
    })
}
const setSelectionChange = (selection: IUserForm[]) => {
    checked.value = []
    checked.value = selection.map((el: IUserForm) => el.id!)
}

// 搜索框
const queryByName = () => {
    params.username = searchUser.value
    refreshTable()
}

// 根据用户编号获取详细信息
const getUserDetail = (row: any) => {
    const levels = row.roles!.map((item: any) => item.level)
    levels.sort(function (a: number, b: number) { return a - b })
    if (levels[0] >= getLoginUser.value.level) {
        userDrawerRef.value.open(row)
    } else {
        ElMessage.warning('权限等级不足!')
    }
}

const checkbox = (row: any) => {
    const levels = row.roles!.map((item: any) => item.level)
    levels.sort(function (a: number, b: number) { return a - b })
    return levels[0] >= getLoginUser.value.level && !row.builtin
}

const httpRequest = (rawFile: any) => {
    const formData = new FormData()

    formData.append('file', rawFile.file)
    importData(formData).then((res: any) => {
        ElMessage.success('导入成功')
        refreshTable()
    })
}
const updateTable = () => {
    refreshTable()
}
// 导入模板
const handleDropDown = (value: string) => {
    if (value === 'export') {
        const exportParams = {
            username: params.username,
            enable: params.enable
        }
        exportData(exportParams).then((res: any) => {
            downloadBlobFile(
                res,
                '用户列表',
                'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
            )
            ElMessage.success('导出成功')
        })
    }
}
const refreshTable = () => {
    tableListRef.value.refresh()
}
</script>

<template>
  <custom-table-layout>
    <template #header>
      <custom-header-layout>
        <template #left>
          <span>用户管理</span>
        </template>
        <template #right>
          <el-button type="primary" @click="addUser" style="margin-right: 10px">添加用户</el-button>
          <el-dropdown @command="handleDropDown">
            <el-button text bg><el-icon>
                <MoreFilled />
              </el-icon></el-button>
            <template #dropdown>
              <el-dropdown-menu>
                <el-upload ref="uploadRef" class="upload-demo" accept=".xlsx" :file-list="file" :show-file-list="false"
                  :http-request="httpRequest" action="none">
                  <el-dropdown-item disabled command="import" :icon="Upload">批量导入</el-dropdown-item>
                </el-upload>
                <el-dropdown-item command="export" :icon="Download">
                  <span>批量导出</span>
                </el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
        </template>
      </custom-header-layout>
    </template>
    <template #toolbar>
      <custom-table-toolbar>
        <template #left>
          <el-input @change="queryByName" :prefix-icon="Search" clearable placeholder="请输入用户账号" v-model="searchUser" />
          <el-select v-model="params.enable" placeholder="启/停" clearable @change="updateTable">
            <el-option label="启用" :value="true"></el-option>
            <el-option label="停用" :value="false"></el-option>
          </el-select>
        </template>
      </custom-table-toolbar>
    </template>
    <template #table>
      <custom-table ref="tableListRef" model="api" :api="queryUserList" :params="params"
        @selection-change="setSelectionChange" :selection="false">
        <el-table-column type="selection" fixed='left' width="50" align="center" :selectable="checkbox" />
        <el-table-column label="用户账号" width="240" class-name="link" fixed="left" show-overflow-tooltip>
          <template #default="scope">
            <span @click="getUserDetail(scope.row)">{{ scope.row.username }}</span>
          </template>
        </el-table-column>
        <el-table-column label="用户昵称" property="nickname" width="260" fixed="left" show-overflow-tooltip />
        <el-table-column label="用户角色" width="370" show-overflow-tooltip>
          <template #default="scope">
            <custom-tag-group>
              <el-tag v-for="role in scope.row.roles" :key="role.id">{{ role.name }}</el-tag>
            </custom-tag-group>
          </template>
        </el-table-column>
        <el-table-column label="手机号码" width="160" class-name="number" prop="phone" />
        <el-table-column label="电子邮箱" width="220" prop="email" />
        <el-table-column label="用户类型" width="120">
          <template #default="scope">
            <el-tag v-if="scope.row.builtin" type="info">内置</el-tag>
            <el-tag v-else>自定义</el-tag>
          </template>
        </el-table-column>
        <el-table-column label="登录时间" width="220">
          <template #default="scope">
            <span v-if="scope.row.loginTime">{{
              dayjs(scope.row.loginTime).format("YYYY-MM-DD HH:mm:ss")
            }}</span>
            <span v-else></span>
          </template>
        </el-table-column>
        <el-table-column label="登录IP" property="loginIp" width="220" />
        <el-table-column label="用户状态" width="130">
          <template #default="scope">
            <span class="status success" v-if="scope.row.enable">启用</span>
            <span class="status pending" v-else>停用</span>
          </template>
        </el-table-column>
        <template #tools>
          <el-button :disabled="checked.length <= 0" @click="onBatchDeletion" :loading="loading">批量删除</el-button>
        </template>
      </custom-table>
    </template>
  </custom-table-layout>

  <create-user-dialog ref="createUserDialogRef" @refresh-table="refreshTable"></create-user-dialog>
  <user-drawer ref="userDrawerRef" @update-table="refreshTable"></user-drawer>
</template>
<style lang="scss" scoped>
.sys-table-header {
  width: 100%;
  display: flex;
  align-items: center;

  &-left {
    display: flex;
    align-items: center;

    .title {
      margin-right: 14px;
      font-weight: 500;
      font-size: 16px;
      color: #000;
    }

    p {
      color: #6b778c;
      font-size: 16px;

      span {
        color: #172b4d;
        padding: 5px;
      }
    }
  }

  &-right {
    margin-left: auto;
    display: flex;
    align-items: center;
    column-gap: 12px;

    .el-select {
      width: 120px;

      .el-input {
        height: 32px;

        .el-input__wrapper {
          background: #dfe1e6;
        }
      }
    }
  }

  .header-msg {
    line-height: 32px;
    flex-shrink: 0;
    padding-right: 20px;
    font-weight: 600;

    span {
      color: #0052cc;
    }
  }

  .header-btn-group {
    .el-button {
      display: inline-flex;
      align-items: center;
      font-size: 14px;

      color: #344563;
      margin-right: 10px;
    }
  }
}

.status {
  padding-left: 16px;

  &::after {
    content: "";
    position: absolute;
    left: 4px;
    top: calc(50% - 4px);
    width: 8px;
    height: 8px;
    margin: 0 8px;
    border-radius: 50%;
  }
}

.success {
  &::after {
    background-color: var(--el-color-success);
  }
}

.pending {
  &::after {
    background-color: var(--el-color-info);
  }
}

.warning {
  &::after {
    background-color: var(--el-color-danger);
  }
}
</style>
@/stores/userStore