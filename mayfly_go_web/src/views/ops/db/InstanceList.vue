<template>
    <div class="db-list">
        <page-table
            ref="pageTableRef"
            :page-api="dbApi.instances"
            :data-handler-fn="handleData"
            :searchItems="searchItems"
            v-model:query-form="query"
            :show-selection="true"
            v-model:selection-data="state.selectionData"
            :columns="columns"
            lazy
        >
            <template #tableHeader>
                <el-button v-auth="perms.saveInstance" type="primary" icon="plus" @click="editInstance(false)">添加</el-button>
                <el-button v-auth="perms.delInstance" :disabled="selectionData.length < 1" @click="deleteInstance()" type="danger" icon="delete"
                    >删除</el-button
                >
            </template>

            <template #tagPath="{ data }">
                <ResourceTags :tags="data.tags" />
            </template>

            <template #authCert="{ data }">
                <ResourceAuthCert v-model:select-auth-cert="data.selectAuthCert" :auth-certs="data.authCerts" />
            </template>

            <template #type="{ data }">
                <el-tooltip :content="getDbDialect(data.type).getInfo().name" placement="top">
                    <SvgIcon :name="getDbDialect(data.type).getInfo().icon" :size="20" />
                </el-tooltip>
            </template>

            <template #action="{ data }">
                <el-button @click="showInfo(data)" link>详情</el-button>
                <el-button v-if="actionBtns[perms.saveInstance]" @click="editInstance(data)" type="primary" link>编辑</el-button>
                <el-button v-if="actionBtns[perms.saveDb]" @click="editDb(data)" type="primary" link>库配置</el-button>
            </template>
        </page-table>

        <el-dialog v-model="infoDialog.visible" title="详情">
            <el-descriptions :column="3" border>
                <el-descriptions-item :span="2" label="名称">{{ infoDialog.data.name }}</el-descriptions-item>
                <el-descriptions-item :span="1" label="id">{{ infoDialog.data.id }}</el-descriptions-item>
                <el-descriptions-item :span="2" label="主机">{{ infoDialog.data.host }}</el-descriptions-item>
                <el-descriptions-item :span="1" label="端口">{{ infoDialog.data.port }}</el-descriptions-item>

                <el-descriptions-item :span="1" label="类型">{{ infoDialog.data.type }}</el-descriptions-item>

                <el-descriptions-item :span="3" label="连接参数">{{ infoDialog.data.params }}</el-descriptions-item>
                <el-descriptions-item :span="3" label="备注">{{ infoDialog.data.remark }}</el-descriptions-item>

                <el-descriptions-item :span="3" label="SSH隧道">{{ infoDialog.data.sshTunnelMachineId > 0 ? '是' : '否' }} </el-descriptions-item>

                <el-descriptions-item :span="2" label="创建时间">{{ dateFormat(infoDialog.data.createTime) }} </el-descriptions-item>
                <el-descriptions-item :span="1" label="创建者">{{ infoDialog.data.creator }}</el-descriptions-item>

                <el-descriptions-item :span="2" label="更新时间">{{ dateFormat(infoDialog.data.updateTime) }} </el-descriptions-item>
                <el-descriptions-item :span="1" label="修改者">{{ infoDialog.data.modifier }}</el-descriptions-item>
            </el-descriptions>
        </el-dialog>

        <instance-edit
            @val-change="search()"
            :title="instanceEditDialog.title"
            v-model:visible="instanceEditDialog.visible"
            v-model:data="instanceEditDialog.data"
        ></instance-edit>

        <instance-db-conf :title="dbEditDialog.title" v-model:visible="dbEditDialog.visible" :instance="dbEditDialog.instance" />
    </div>
</template>

<script lang="ts" setup>
import { defineAsyncComponent, onMounted, reactive, ref, Ref, toRefs } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { dbApi } from './api';
import { dateFormat } from '@/common/utils/date';
import PageTable from '@/components/pagetable/PageTable.vue';
import { TableColumn } from '@/components/pagetable';
import { hasPerms } from '@/components/auth/auth';
import SvgIcon from '@/components/svgIcon/index.vue';
import { getDbDialect } from './dialect';
import { SearchItem } from '@/components/SearchForm';
import ResourceAuthCert from '../component/ResourceAuthCert.vue';
import ResourceTags from '../component/ResourceTags.vue';
import { getTagPathSearchItem } from '../component/tag';
import { TagResourceTypeEnum } from '@/common/commonEnum';

const InstanceEdit = defineAsyncComponent(() => import('./InstanceEdit.vue'));
const InstanceDbConf = defineAsyncComponent(() => import('./InstanceDbConf.vue'));

const props = defineProps({
    lazy: {
        type: [Boolean],
        default: false,
    },
});

const perms = {
    saveInstance: 'db:instance:save',
    delInstance: 'db:instance:del',
    saveDb: 'db:save',
};

const searchItems = [getTagPathSearchItem(TagResourceTypeEnum.Db.value), SearchItem.input('code', '编号'), SearchItem.input('name', '名称')];

const columns = ref([
    TableColumn.new('tags[0].tagPath', '关联标签').isSlot('tagPath').setAddWidth(20),
    TableColumn.new('name', '名称'),
    TableColumn.new('type', '类型').isSlot().setAddWidth(-15).alignCenter(),
    TableColumn.new('host', 'host:port').setFormatFunc((data: any) => `${data.host}:${data.port}`),
    TableColumn.new('authCerts[0].username', '授权凭证').isSlot('authCert').setAddWidth(10),
    TableColumn.new('params', '连接参数'),
    TableColumn.new('remark', '备注'),
    TableColumn.new('code', '编号'),
]);

// 该用户拥有的的操作列按钮权限
const actionBtns = hasPerms(Object.values(perms));
const actionColumn = TableColumn.new('action', '操作').isSlot().setMinWidth(180).fixedRight().alignCenter();
const pageTableRef: Ref<any> = ref(null);

const state = reactive({
    row: {},
    dbId: 0,
    db: '',
    /**
     * 选中的数据
     */
    selectionData: [],
    /**
     * 查询条件
     */
    query: {
        name: null,
        tagPath: '',
        pageNum: 1,
        pageSize: 0,
    },
    infoDialog: {
        visible: false,
        data: null as any,
    },
    instanceEditDialog: {
        visible: false,
        data: null as any,
        title: '新增数据库实例',
    },
    dbEditDialog: {
        visible: false,
        instance: null as any,
        title: '新增数据库实例',
    },
});

const { selectionData, query, infoDialog, instanceEditDialog, dbEditDialog } = toRefs(state);

onMounted(async () => {
    if (Object.keys(actionBtns).length > 0) {
        columns.value.push(actionColumn);
    }
    if (!props.lazy) {
        search();
    }
});

const search = (tagPath: string = '') => {
    if (tagPath) {
        state.query.tagPath = tagPath;
    }
    pageTableRef.value.search();
};

const handleData = (res: any) => {
    const dataList = res.list;
    // 赋值授权凭证
    for (let x of dataList) {
        x.selectAuthCert = x.authCerts[0];
    }
    return res;
};

const showInfo = (info: any) => {
    state.infoDialog.data = info;
    state.infoDialog.visible = true;
};

const editInstance = async (data: any) => {
    if (!data) {
        state.instanceEditDialog.data = null;
        state.instanceEditDialog.title = '新增数据库实例';
    } else {
        state.instanceEditDialog.data = data;
        state.instanceEditDialog.title = '修改数据库实例';
    }
    state.instanceEditDialog.visible = true;
};

const deleteInstance = async () => {
    try {
        await ElMessageBox.confirm(`确定删除数据库实例【${state.selectionData.map((x: any) => x.name).join(', ')}】?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning',
        });
        await dbApi.deleteInstance.request({ id: state.selectionData.map((x: any) => x.id).join(',') });
        ElMessage.success('删除成功');
        search();
    } catch (err) {
        //
    }
};

const editDb = (data: any) => {
    state.dbEditDialog.instance = data;
    state.dbEditDialog.title = `配置 "${data.name}" 数据库`;
    state.dbEditDialog.visible = true;
};

defineExpose({ search });
</script>
<style lang="scss"></style>
