<template>
    <div>
        <div class="app-status" v-if="data.isExist">
            <el-card>
                <div class="flex items-center">
                    <div>
                        <el-tag effect="dark" type="success">{{ data.app }}</el-tag>
                    </div>
                    <div>
                        <Status class="status-content" :key="refresh" :status="data.status"></Status>
                    </div>
                    <div>
                        <el-tag class="status-content">{{ $t('app.version') }}:{{ data.version }}</el-tag>
                    </div>
                    <div>
                        <span class="buttons">
                            <el-button
                                type="primary"
                                v-if="data.status != 'Running'"
                                link
                                @click="onOperate('start')"
                                :disabled="data.status === 'Installing'"
                            >
                                {{ $t('app.start') }}
                            </el-button>
                            <el-button type="primary" v-if="data.status === 'Running'" link @click="onOperate('stop')">
                                {{ $t('app.stop') }}
                            </el-button>
                            <el-divider direction="vertical" />
                            <el-button
                                type="primary"
                                link
                                :disabled="data.status === 'Installing'"
                                @click="onOperate('restart')"
                            >
                                {{ $t('app.restart') }}
                            </el-button>
                            <el-divider direction="vertical" />
                            <el-button
                                type="primary"
                                link
                                v-if="data.app === 'OpenResty'"
                                @click="onOperate('reload')"
                                :disabled="data.status !== 'Running'"
                            >
                                {{ $t('app.reload') }}
                            </el-button>
                            <el-divider v-if="data.app === 'OpenResty'" direction="vertical" />
                            <el-button type="primary" @click="setting" link :disabled="data.status === 'Installing'">
                                {{ $t('commons.button.set') }}
                            </el-button>
                            <el-divider v-if="data.app === 'OpenResty'" direction="vertical" />
                            <el-button
                                v-if="data.app === 'OpenResty'"
                                type="primary"
                                @click="clear"
                                link
                                :disabled="
                                    data.status === 'Installing' ||
                                    (data.status !== 'Running' && data.app === 'OpenResty')
                                "
                            >
                                {{ $t('nginx.clearProxyCache') }}
                            </el-button>
                        </span>
                    </div>

                    <div class="ml-5" v-if="key === 'openresty' && (httpPort != 80 || httpsPort != 443)">
                        <el-tooltip
                            effect="dark"
                            :content="$t('website.openrestyHelper', [httpPort, httpsPort])"
                            placement="top-start"
                        >
                            <el-alert
                                :title="$t('app.checkTitle')"
                                :closable="false"
                                type="warning"
                                show-icon
                                class="h-8"
                            />
                        </el-tooltip>
                    </div>
                </div>
            </el-card>
        </div>
    </div>
</template>
<script lang="ts" setup>
import { CheckAppInstalled, InstalledOp } from '@/api/modules/app';
import { onMounted, reactive, ref } from 'vue';
import Status from '@/components/status/index.vue';
import { ElMessageBox } from 'element-plus';
import i18n from '@/lang';
import { MsgSuccess } from '@/utils/message';
import { ClearNginxCache } from '@/api/modules/nginx';

const props = defineProps({
    appKey: {
        type: String,
        default: 'openresty',
    },
    appName: {
        type: String,
        default: '',
    },
});

let key = ref('');
let name = ref('');

let data = ref({
    app: '',
    version: '',
    status: '',
    lastBackupAt: '',
    appInstallId: 0,
    isExist: false,
    containerName: '',
});
let operateReq = reactive({
    installId: 0,
    operate: '',
});
let refresh = ref(1);
const httpPort = ref(0);
const httpsPort = ref(0);

const em = defineEmits(['setting', 'isExist', 'before', 'after', 'update:loading', 'update:maskShow']);
const setting = () => {
    em('setting', false);
};

const onCheck = async (key: any, name: any) => {
    await CheckAppInstalled(key, name)
        .then((res) => {
            data.value = res.data;
            em('isExist', res.data);
            em('update:maskShow', res.data.status !== 'Running');
            operateReq.installId = res.data.appInstallId;
            httpPort.value = res.data.httpPort;
            httpsPort.value = res.data.httpsPort;
            refresh.value++;
        })
        .catch(() => {
            em('isExist', false);
            refresh.value++;
        });
};

const clear = () => {
    ElMessageBox.confirm(i18n.global.t('nginx.clearProxyCacheWarn'), i18n.global.t('nginx.clearProxyCache'), {
        confirmButtonText: i18n.global.t('commons.button.confirm'),
        cancelButtonText: i18n.global.t('commons.button.cancel'),
    }).then(async () => {
        await ClearNginxCache();
        MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
    });
};

const onOperate = async (operation: string) => {
    em('update:maskShow', false);
    operateReq.operate = operation;
    ElMessageBox.confirm(
        i18n.global.t('app.operatorHelper', [i18n.global.t('app.' + operation)]),
        i18n.global.t('app.' + operation),
        {
            confirmButtonText: i18n.global.t('commons.button.confirm'),
            cancelButtonText: i18n.global.t('commons.button.cancel'),
            type: 'info',
        },
    )
        .then(() => {
            em('update:maskShow', true);
            em('update:loading', true);
            em('before');
            InstalledOp(operateReq)
                .then(() => {
                    em('update:loading', false);
                    MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
                    onCheck(key.value, name.value);
                    em('after');
                })
                .catch(() => {
                    em('update:loading', false);
                });
        })
        .catch(() => {
            em('update:maskShow', true);
        });
};

onMounted(() => {
    key.value = props.appKey;
    name.value = props.appName;
    onCheck(key.value, name.value);
});

defineExpose({
    onCheck,
});
</script>
