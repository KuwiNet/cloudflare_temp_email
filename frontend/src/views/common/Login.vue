<script setup>
import { ref, onMounted, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'
import { NewLabelOutlined, EmailOutlined } from '@vicons/material'

import AdminContact from '../common/AdminContact.vue'
import Turnstile from '../../components/Turnstile.vue'

import { useGlobalState } from '../../store'
import { api } from '../../api'
import { getRouterPathWithLang } from '../../utils'

const props = defineProps({
    bindUserAddress: {
        type: Function,
        default: async () => { await api.bindUserAddress(); },
        required: true
    },
    newAddressPath: {
        type: Function,
        default: async (address_name, domain, cf_token) => {
            return await api.fetch("/api/new_address", {
                method: "POST",
                body: JSON.stringify({
                    name: address_name,
                    domain: domain,
                    cf_token: cf_token,
                }),
            });
        },
        required: true
    },
})

const message = useMessage()
const notification = useNotification()
const router = useRouter()

const {
    jwt, loading, openSettings,
    showAddressCredential, userSettings
} = useGlobalState()

const tabValue = ref('signin')
const credential = ref('')
const emailName = ref("")
const emailDomain = ref("")
const cfToken = ref("")

const login = async () => {
    if (!credential.value) {
        message.error(t('credentialInput'));
        return;
    }
    try {
        jwt.value = credential.value;
        await api.getSettings();
        try {
            await props.bindUserAddress();
        } catch (error) {
            message.error(`${t('bindUserAddressError')}: ${error.message}`);
        }
        await router.push(getRouterPathWithLang("/", locale.value));
    } catch (error) {
        message.error(error.message || "error");
    }
}

const { locale, t } = useI18n({
    messages: {
        en: {
            login: 'Login',
			loginAndBind: 'Login and Bind',
            pleaseGetNewEmail: 'Please login or click "Get New Email" button to get a new email address',
            getNewEmail: 'Create New Email',
            getNewEmailTip1: 'Please input the email you want to use. only allow: ',
            getNewEmailTip2: 'Levaing it blank will generate a random email address.',
            getNewEmailTip3: 'You can choose a domain from the dropdown list.',
            getNewEmailTip4: 'After registering and logging in, you can use the [abcd.pm] domain name.',
            getNewEmailTip5: 'Note: The domain names [aurl.pp.ua],[kuwi.pp.ua],[yotu.pp.ua] and [gede.eu.org] only support receiving emails, not sending emails.',
            credential: 'Email Address Credential',
            ok: 'OK',
            generateName: 'Generate Fake Name',
            help: 'Help',
            credentialInput: 'Please input the Mail Address Credential',
            bindUserInfo: 'Logged in user, login without binding email or create new email address will bind to current user',
            bindUserAddressError: 'Error when bind email address to user',
        },
        zh: {
            login: '登录',
			loginAndBind: '登录并绑定',
            pleaseGetNewEmail: '请"登录"或点击 "注册新邮箱" 按钮来获取一个新的邮箱地址',
            getNewEmail: '创建新邮箱',
            getNewEmailTip1: '请输入你想要使用的邮箱地址, 只允许: ',
            getNewEmailTip2: '留空将会生成一个随机的邮箱地址。',
            getNewEmailTip3: '你可以从下拉列表中选择一个域名。',
            getNewEmailTip4: '注册并登陆后，方可使用 [abcd.pm] 域名。',
            getNewEmailTip5: '注：[aurl.pp.ua]、[kuwi.pp.ua]、[yotu.pp.ua]、[gede.eu.org] 域名仅支持接收邮件，不支持发送邮件。',
            credential: '邮箱地址凭据',
            ok: '确定',
            generateName: '生成随机名字',
            help: '帮助',
            credentialInput: '请输入邮箱地址凭据',
            bindUserInfo: '已登录用户, 登录未绑定邮箱或创建新邮箱地址将绑定到当前用户',
            bindUserAddressError: '绑定邮箱地址到用户时错误',
        }
    }
});
const loginAndBindTag = computed(() => {
    if (userSettings.value.user_email) {
        return t('loginAndBind')
    }
    return t('login')
})
const addressRegex = computed(() => {
    try {
        if (openSettings.value.addressRegex) {
            return new RegExp(openSettings.value.addressRegex, 'g');
        }
    } catch (error) {
        console.error(error);
        message.error(`Invalid addressRegex: ${openSettings.value.addressRegex}`);
    }
    return /[^a-z0-9]/g;
});
const generateNameLoading = ref(false);
const generateName = async () => {
    try {
        generateNameLoading.value = true;
        const { faker } = await import('https://esm.sh/@faker-js/faker');
        emailName.value = faker.internet.email()
            .split('@')[0]
            .replace(/\s+/g, '.')
            .replace(/\.{2,}/g, '.')
            .replace(addressRegex.value, '')
            .toLowerCase();
        if (emailName.value.length > openSettings.value.maxAddressLen) {
            emailName.value = emailName.value.slice(0, openSettings.value.maxAddressLen);
        }
    } catch (error) {
        message.error(error.message || "error");
    } finally {
        generateNameLoading.value = false;
    }
};

const newEmail = async () => {
    try {
        const res = await props.newAddressPath(
            emailName.value,
            emailDomain.value,
            cfToken.value
        );
        jwt.value = res["jwt"];
        await api.getSettings();
        await router.push(getRouterPathWithLang("/", locale.value));
        showAddressCredential.value = true;
        try {
            await props.bindUserAddress();
        } catch (error) {
            message.error(`${t('bindUserAddressError')}: ${error.message}`);
        }
    } catch (error) {
        message.error(error.message || "error");
    }
};

const addressPrefix = computed(() => {
    // if user has role, return role prefix
    if (userSettings.value?.user_role) {
        return userSettings.value.user_role.prefix || "";
    }
    // if user has no role, return default prefix
    return openSettings.value.prefix;
});

const domainsOptions = computed(() => {
    // if user has role, return role domains
    if (userSettings.value.user_role) {
        const allDomains = userSettings.value.user_role.domains;
        if (!allDomains) return openSettings.value.domains;
        return openSettings.value.domains.filter((domain) => {
            return allDomains.includes(domain.value);
        });
    }
    // if user has no role, return default domains
    if (!openSettings.value.defaultDomains) {
        return openSettings.value.domains;
    }
    // if user has no role and no default domains, return all domains
    return openSettings.value.domains.filter((domain) => {
        return openSettings.value.defaultDomains.includes(domain.value);
    });
});

const showNewAddressTab = computed(() => {
    if (openSettings.value.disableAnonymousUserCreateEmail
        && !userSettings.value.user_email
    ) {
        return false;
    }
    return openSettings.value.enableUserCreateEmail;
});

onMounted(async () => {
    if (!openSettings.value.domains || openSettings.value.domains.length === 0) {
        await api.getOpenSettings(message, notification);
    }
    emailDomain.value = domainsOptions.value ? domainsOptions.value[0]?.value : "";
});
</script>

<template>
    <div>
        <n-alert v-if="userSettings.user_email" :show-icon="false" :bordered="false" closable>
            <span>{{ t('bindUserInfo') }}</span>
        </n-alert>
        <n-tabs v-if="openSettings.fetched" v-model:value="tabValue" size="large" justify-content="space-evenly">
            <n-tab-pane name="signin" :tab="loginAndBindTag">
                <n-form>
                    <n-form-item-row :label="t('credential')" required>
                        <n-input v-model:value="credential" type="textarea" :autosize="{ minRows: 3 }" />
                    </n-form-item-row>
                    <n-button @click="login" :loading="loading" type="primary" block secondary strong>
                        <template #icon>
                            <n-icon :component="EmailOutlined" />
                        </template>
                        {{ loginAndBindTag }}
                    </n-button>
                    <n-button v-if="showNewAddressTab" @click="tabValue = 'register'" block secondary strong>
                        <template #icon>
                            <n-icon :component="NewLabelOutlined" />
                        </template>
                        {{ t('getNewEmail') }}
                    </n-button>
                </n-form>
            </n-tab-pane>
            <n-tab-pane v-if="showNewAddressTab" name="register" :tab="t('getNewEmail')">
                <n-spin :show="generateNameLoading">
                    <n-form>
                        <span>
                            <p>{{ t("getNewEmailTip1") + addressRegex.source }}</p>
                            <p>{{ t("getNewEmailTip2") }}</p>
                            <p>{{ t("getNewEmailTip3") }}</p>
                            <p>{{ t("getNewEmailTip4") }}</p>
                            <p>{{ t("getNewEmailTip5") }}</p>
                        </span>
                        <n-button @click="generateName" style="margin-bottom: 10px;">
                            {{ t('generateName') }}
                        </n-button>
                        <n-input-group>
                            <n-input-group-label v-if="addressPrefix">
                                {{ addressPrefix }}
                            </n-input-group-label>
                            <n-input v-model:value="emailName" show-count :minlength="openSettings.minAddressLen"
                                :maxlength="openSettings.maxAddressLen" />
                            <n-input-group-label>@</n-input-group-label>
                            <n-select v-model:value="emailDomain" :consistent-menu-width="false"
                                :options="domainsOptions" />
                        </n-input-group>
                        <Turnstile v-model:value="cfToken" />
                        <n-button type="primary" block secondary strong @click="newEmail" :loading="loading">
                            <template #icon>
                                <n-icon :component="NewLabelOutlined" />
                            </template>
                            {{ t('getNewEmail') }}
                        </n-button>
                    </n-form>
                </n-spin>
            </n-tab-pane>
            <n-tab-pane name="help" :tab="t('help')">
                <n-alert :show-icon="false" :bordered="false">
                    <span>{{ t('pleaseGetNewEmail') }}</span>
                </n-alert>
                <AdminContact />
            </n-tab-pane>
        </n-tabs>
    </div>
</template>


<style scoped>
.n-alert {
    margin-top: 10px;
    margin-bottom: 10px;
    text-align: center;
}

.n-form .n-button {
    margin-top: 10px;
}

.n-form {
    text-align: left;
}
</style>
