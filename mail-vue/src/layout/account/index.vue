<template>
  <div class="account-box">
    <div class="head-opt" >
      <Icon v-perm="'account:add'" class="icon add" icon="ion:add-outline" width="23" height="23" @click="add" />
      <Icon class="icon refresh" icon="ion:reload" width="18" height="18"  @click="refresh" />
    </div>
    <el-scrollbar class="scrollbar">
      <div v-infinite-scroll="getAccountList" :infinite-scroll-distance="600"  :infinite-scroll-immediate="false">
        <el-card class="item" :class="itemBg(item.accountId)" v-for="item in accounts" :key="item.accountId" @click="changeAccount(item)">
          <div class="account" >
            {{ item.email }}
          </div>
          <div class="opt">
            <div class="send-email" @click.stop>
              <Icon  icon="eva:email-fill" width="22" height="22" color="#fbbd08" />
            </div>
            <div class="settings" @click.stop>
              <Icon icon="streamline-ultimate-color:copy-paste-1" width="19" height="19" @click.stop="copyAccount(item.email)"/>
              <Icon icon="fluent:settings-24-filled" width="21" height="21" color="#909399" v-if="showNullSetting(item)" />
              <el-dropdown v-else>
                <Icon icon="fluent:settings-24-filled"  width="21" height="21" color="#909399" />
                <template #dropdown >
                  <el-dropdown-menu >
                    <el-dropdown-item v-if="hasPerm('email:send')" @click="openSetName(item)">{{$t('rename')}}</el-dropdown-item>
                    <el-dropdown-item v-if="item.accountId !== userStore.user.accountId && hasPerm('account:delete')" @click="remove(item)">{{$t('delete')}}</el-dropdown-item>
                  </el-dropdown-menu>
                </template>
              </el-dropdown>
            </div>
          </div>
        </el-card>

        <!-- Initial Loading Skeleton -->
        <template v-if="loading">
          <el-skeleton v-for="i in 3" :key="i"  animated>
            <template #template>
              <el-card class="item">
                <el-skeleton-item variant="p" style="width: 70%; height: 20px; margin-bottom: 20px" />
                <div style="display: flex; justify-content: space-between">
                  <el-skeleton-item variant="text" style="width: 20px" />
                  <el-skeleton-item variant="text" style="width: 20px" />
                </div>
              </el-card>
            </template>
          </el-skeleton>
        </template>

        <!-- Follow Loading Skeleton -->
        <template v-if="accounts.length > 0 && !noLoading">
          <el-skeleton animated>
            <template #template>
              <el-card class="item">
                <el-skeleton-item variant="p" style="width: 70%; height: 20px; margin-bottom: 20px" />
                <div style="display: flex; justify-content: space-between">
                  <el-skeleton-item variant="text" style="width: 20px" />
                  <el-skeleton-item variant="text" style="width: 20px" />
                </div>
              </el-card>
            </template>
          </el-skeleton>
        </template>

        <div class="noLoading" v-if="noLoading && accounts.length > 0">
          <div>{{$t('noMoreData')}}</div>
        </div>
        <div class="empty" v-if="noLoading && accounts.length === 0">
          <el-empty :description="$t('noMessagesFound')" />
        </div>
      </div>

    </el-scrollbar>
    <el-dialog v-model="showAdd" :title="$t('addAccount')" >
        <div class="container">
          <el-input v-model="addForm.email" ref="addRef" type="text" :placeholder="$t('emailAccount')" autocomplete="off">
            <template #append>
              <div  @click.stop="openSelect">
                <el-select
                    ref="mySelect"
                    v-model="addForm.suffix"
                    :placeholder="$t('select')"
                    class="select"
                >
                  <el-option
                      v-for="item in domainList"
                      :key="item"
                      :label="item"
                      :value="item"
                  />
                </el-select>
                <div style="color: #333">
                  <span >{{addForm.suffix}}</span>
                  <Icon class="setting-icon" icon="mingcute:down-small-fill" width="20" height="20" />
                </div>
              </div>
            </template>
          </el-input>
          <el-button class="btn" type="primary" @click="submit" :loading="addLoading"
          >{{$t('add')}}
          </el-button>
        </div>
      <div
          class="add-email-turnstile"
          :class="verifyShow ? 'turnstile-show' : 'turnstile-hide'"
          :data-sitekey="settingStore.settings.siteKey"
          data-callback="onTurnstileSuccess"
          data-error-callback="onTurnstileError"
      >
        <span style="font-size: 12px;color: #F56C6C" v-if="botJsError">人机验证模块加载失败,请刷新浏览器</span>
      </div>
    </el-dialog>
    <el-dialog v-model="setNameShow" :title="$t('changeUserName')" >
      <div class="container">
        <el-input v-model="accountName" type="text" :placeholder="$t('username')" autocomplete="off">
        </el-input>
        <el-button class="btn" type="primary" @click="setName" :loading="setNameLoading"
        >{{$t('save')}}
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script setup>
import {Icon} from "@iconify/vue";
import {nextTick, reactive, ref, watch} from "vue";
import {accountList, accountAdd, accountDelete, accountSetName} from "@/request/account.js";
import {isEmail} from "@/utils/verify-utils.js";
import {useSettingStore} from "@/store/setting.js";
import {useAccountStore} from "@/store/account.js";
import {useUserStore} from "@/store/user.js";
import { hasPerm } from "@/perm/perm.js"
import {useI18n} from "vue-i18n";

const { t } = useI18n();
const userStore = useUserStore();
const accountStore = useAccountStore();
const settingStore = useSettingStore();
const showAdd = ref(false)
const addLoading = ref(false);
const domainList = settingStore.domainList
const accounts = reactive([])
const noLoading = ref(false)
const loading = ref(false)
const followLoading = ref(false);
const verifyShow = ref(false)
const setNameShow = ref(false)
const setNameLoading = ref(false)
const accountName = ref(null)
const addRef = ref({})
let account = null
let turnstileId = null
const botJsError = ref(false)
let verifyToken = ''
const addForm = reactive({
  email: '',
  suffix: settingStore.domainList[0]
})
const queryParams = {
  accountId: 0,
  size: 20
}

const mySelect = ref()

if (hasPerm('account:query')) {
  getAccountList()
}

watch(() => accountStore.changeUserAccountName, () => {
  accounts[0].name = accountStore.changeUserAccountName
})


const openSelect = () => {
  mySelect.value.toggleMenu()
}

window.onTurnstileError = (e) => {
  console.log('人机验加载失败')
  nextTick(() => {
    if (!turnstileId) {
      turnstileId = window.turnstile.render('.register-turnstile')
    } else {
      window.turnstile.reset(turnstileId);
    }
  })
};

window.onTurnstileSuccess = (token) => {
  verifyToken = token;
};

function setName() {

  let name = accountName.value

  if (name === account.name) {
    setNameShow.value = false
    return
  }

  if (!name) {
    ElMessage({
      message: t('emptyUserNameMsg'),
      type: 'error',
      plain: true,
    })
    return;
  }

  setNameLoading.value = true
  accountSetName(account.accountId,name).then(() => {
    account.name = name
    setNameShow.value = false

    if (account.accountId === userStore.user.accountId) {
      userStore.user.name = name
    }

    ElMessage({
      message: t('changSuccessMsg'),
      type: "success",
      plain: true
    })
  }).finally(()=> {
    setNameLoading.value = false
  })
}

function openSetName (accountItem) {
  accountName.value = accountItem.name
  account = accountItem
  setNameShow.value = true
}

function showNullSetting(item) {
  return !hasPerm('email:send') && !(item.accountId !== userStore.user.accountId && hasPerm('account:delete'))
}

function itemBg(accountId) {
  return accountStore.currentAccountId === accountId ? 'item-choose' : ''
}

function remove(account) {
  ElMessageBox.confirm(t('delConfirm',{msg: account.email}), {
    confirmButtonText: t('confirm'),
    cancelButtonText: t('cancel'),
    type: 'warning'
  }).then(() => {
    accountDelete(account.accountId).then(() => {
      const index = accounts.findIndex(item => item.accountId === account.accountId);
      accounts.splice(index, 1);
      if (accounts.length < queryParams.size) {
        getAccountList()
      }
      ElMessage({
        message: t('delSuccessMsg'),
        type: 'success',
        plain: true,
      })
    })
  });
}

function refresh() {
  if (loading.value) {
    return
  }
  loading.value = false
  followLoading.value = false
  noLoading.value = false
  queryParams.accountId = 0
  accounts.splice(0, accounts.length)
  getAccountList()
}
function changeAccount(account) {
  accountStore.currentAccountId = account.accountId
  accountStore.currentAccount = account
}

function add() {
  showAdd.value = true
  setTimeout(() => {
    addRef.value.focus()
  },100)
}

async function copyAccount(account) {
  try {
    await navigator.clipboard.writeText(account);
    ElMessage({
      message: t('copySuccessMsg'),
      type: 'success',
      plain: true,
    })
  } catch (err) {
    console.error(`${t('copyFailMsg')}:`, err);
    ElMessage({
      message: t('copyFailMsg'),
      type: 'error',
      plain: true,
    })
  }
}

function getAccountList() {

  if (loading.value || followLoading.value || noLoading.value) return;

  if (accounts.length === 0) {
    loading.value = true
  }else {
    followLoading.value = true
  }

  accountList(queryParams.accountId,queryParams.size).then(list => {
    if (list.length < queryParams.size) {
      noLoading.value = true
    }
    if (accounts.length === 0) {
      accountStore.currentAccount = list[0].accountId
    }
    queryParams.accountId = list.at(-1).accountId
    accounts.push(...list)

    loading.value = false
    followLoading.value = false
  }).catch(() => {
    loading.value = false
    followLoading.value = false
  })
}


function submit()  {

  if (!addForm.email){
    ElMessage({
      message: t('emptyEmailMsg'),
      type: "error",
      plain: true
    })
    return
  }

  if (!isEmail(addForm.email+addForm.suffix)) {
    ElMessage({
      message: t('notEmailMsg'),
      type: "error",
      plain: true
    })
    return
  }
  if (!verifyToken && (settingStore.settings.addEmailVerify === 0 || (settingStore.settings.addEmailVerify === 2 && settingStore.settings.addVerifyOpen))) {
    if (!verifyShow.value) {
      verifyShow.value = true
      nextTick(() => {
        if (!turnstileId) {
          try {
            turnstileId = window.turnstile.render('.add-email-turnstile')
          } catch (e) {
            botJsError.value = true
            console.log('人机验证js加载失败')
          }
        } else {
          window.turnstile.reset('.add-email-turnstile')
        }
      })
    } else if (!botJsError.value) {
      ElMessage({
        message: t('botVerifyMsg'),
        type: "error",
        plain: true
      })
    }
    return;
  }

  addLoading.value = true
  accountAdd(addForm.email+addForm.suffix,verifyToken).then(account => {
    addLoading.value = false
    showAdd.value = false
    addForm.email = ''
    accounts.push(account)
    verifyToken = ''
    settingStore.settings.addVerifyOpen = account.addVerifyOpen
    ElMessage({
      message: t('addSuccessMsg'),
      type: "success",
      plain: true
    })
    verifyShow.value = false
    userStore.refreshUserInfo()
  }).catch(res => {
    if (res.code === 400) {
      verifyToken = ''
      if (turnstileId) {
        window.turnstile.reset(turnstileId)
      } else {
        nextTick(() => {
          turnstileId = window.turnstile.render('.add-email-turnstile')
        })
      }
      verifyShow.value = true
    }
    addLoading.value = false
  })
}
</script>
<style>
path[fill="#ffdda1"] {
  fill: #ffdd7d;
}
</style>
<style scoped lang="scss">
.account-box {

  border-right: 1px solid var(--el-border-color) !important;
  background-color: #FFF;
  height: 100%;
  overflow: hidden;
  .head-opt {
    display: flex;
    align-items: center;
    height: 38px;
    box-shadow: inset 0 -1px 0 0 rgba(100, 121, 143, 0.12);
    padding-left: 10px;
    padding-right: 10px;
    .icon{
      cursor: pointer;
    }
    .refresh {
      margin-left: 10px;
    }

    .add {
      margin-left: 2px;
    }

    .head-opt:not(.add) .refresh {
      margin-left: 5px;
    }
  }
  .scrollbar {
    width: 100%;
    height: calc(100% - 38px);
    overflow: auto;
    @media (max-width: 767px) {
      height: calc(100% - 98px);
    }
    .empty {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
    }
    .noLoading {
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 10px 0;
      color: gray;
    }
  }
  .btn {
    width: 100%;
    margin-top: 15px;
  }

  .item {
    background-color: #fff;
    border-radius: 8px;
    padding: 12px 10px;
    margin-bottom: 10px;
    margin-left: 10px;
    margin-right: 10px;
    cursor: pointer;
    .account {
      font-weight: 600;
      margin-bottom: 20px;
      color: #333;
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
    }

    .opt {
      display: flex;
      justify-content: space-between;
      font-size: 12px;
      color: #888;
      .settings {
        display: flex;
        align-items: center;
        gap: 10px;
      }
      .send-email {
        display: flex;
        align-items: center;
      }
    }

    :deep(.el-card__body) {
      padding: 0;
    }
  }
  .item:first-child{
    margin-top: 10px  ;
  }

  .item-choose {
    background: var(--el-color-primary-light-8);
  }
}


.setting-icon {
  position: relative;
  top: 6px;
}

:deep(.el-input-group__append) {
  padding: 0 !important;
  padding-left: 8px !important;
  background: #FFFFFF;
}

:deep(.el-dialog) {
  width: 400px !important;
  @media (max-width: 440px) {
    width: calc(100% - 40px) !important;
    margin-right: 20px !important;
    margin-left: 20px !important;
  }
}

.select {
  position: absolute;
  right: 30px;
  width: 100px;
  opacity: 0;
  pointer-events: none;
}

:deep(.el-pagination .el-select) {
  width: 100px;
  background: #FFF;
}

.add-email-turnstile {
  margin-top: 15px;
}

.turnstile-show {
  opacity: 1;
}

.turnstile-hide {
  opacity: 0;
  pointer-events: none;
  position: fixed;
}

</style>