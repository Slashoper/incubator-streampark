<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<script setup lang="ts">
  import { onUnmounted, reactive, unref, ref, onMounted, computed } from 'vue';
  import { AppStateEnum, DeployMode } from '/@/enums/sparkEnum';
  import { useI18n } from '/@/hooks/web/useI18n';
  // import { fetchAppExternalLink } from '/@/api/setting/externalLink';
  import { ExternalLink } from '/@/api/setting/types/externalLink.type';
  // import { useModal } from '/@/components/Modal';
  import { PageWrapper } from '/@/components/Page';
  import { Description, useDescription } from '/@/components/Description';
  import { Icon } from '/@/components/Icon';
  import { useRoute, useRouter } from 'vue-router';
  import { useIntervalFn } from '@vueuse/core';
  import { Divider, Space } from 'ant-design-vue';
  import { handleView } from './utils';
  // import { Button } from '/@/components/Button';
  import { getDescSchema } from './data/detail.data';
  import { fetchSparkConfList } from '/@/api/spark/conf';
  import Mergely from './components/Mergely.vue';
  import DetailTab from './components/DetailTab.vue';
  // import RequestModal from '/@/views/flink/app/components/RequestModal';
  import { createDetailProviderContext } from './hooks/useDetailContext';
  import { useDrawer } from '/@/components/Drawer';
  import { LinkBadge } from '/@/components/LinkBadge';
  import { SparkApplication } from '/@/api/spark/app.type';
  import {
    fetchGetSparkApp,
    fetchSparkBackUps,
    fetchSparkOptionLog,
    fetchSparkYarn,
  } from '/@/api/spark/app';
  import { isNullOrUnDef } from '/@/utils/is';

  defineOptions({
    name: 'ApplicationDetail',
  });
  const route = useRoute();
  const router = useRouter();

  const { t } = useI18n();

  const yarn = ref('');
  const externalLinks = ref<ExternalLink[]>([]);
  const app = reactive<Partial<SparkApplication>>({});
  const detailTabs = reactive({
    showConf: false,
    showSaveOption: false,
    showBackup: false,
    showOptionLog: false,
  });

  createDetailProviderContext({ app });

  const [registerDescription] = useDescription({
    schema: [
      ...(getDescSchema() as any),
      // {
      //   field: 'resetApi',
      //   span: 2,
      //   label: h('div', null, [
      //     t('spark.app.detail.resetApi'),
      //     h(Tooltip, { title: t('spark.app.detail.resetApiToolTip'), placement: 'top' }, () =>
      //       h(Icon, { icon: 'ant-design:question-circle-outlined', class: 'pl-5px', color: 'red' }),
      //     ),
      //   ]),
      //   render: () => [
      //     h(
      //       Button,
      //       {
      //         type: 'primary',
      //         size: 'small',
      //         class: 'mx-3px px-5px',
      //         onClick: () =>
      //           openApiModal(true, {
      //             name: 'flinkStart',
      //             app,
      //           }),
      //       },
      //       () => [t('spark.app.detail.copyStartcURL')],
      //     ),
      //     h(
      //       Button,
      //       {
      //         type: 'primary',
      //         size: 'small',
      //         class: 'mx-3px px-5px',
      //         onClick: () => {
      //           openApiModal(true, {
      //             name: 'flinkCancel',
      //             app,
      //           });
      //         },
      //       },
      //       () => [t('spark.app.detail.copyCancelcURL')],
      //     ),
      //   ],
      // },
    ],
    data: app,
    layout: 'horizontal',
    column: 2,
  });

  const [registerConfDrawer] = useDrawer();
  // const [registerOpenApi, { openModal: openApiModal }] = useModal();

  /* Flink Web UI */
  function handleFlinkView() {
    handleView(app as any, unref(yarn));
  }

  const { pause } = useIntervalFn(
    () => {
      handleGetAppInfo();
    },
    5000,
    { immediateCallback: true },
  );

  async function handleGetAppInfo() {
    if (!route.query.appId) router.back();
    const res = await fetchGetSparkApp({
      id: route.query.appId as string,
    });
    // Get data for the first time
    if (Object.keys(app).length == 0) {
      if (
        !isNullOrUnDef(res.deployMode) &&
        [DeployMode.YARN_CLIENT, DeployMode.YARN_CLUSTER].includes(res.deployMode)
      ) {
        await handleYarn();
      }
      await handleDetailTabs();
    }
    Object.assign(app, res);
  }

  async function handleDetailTabs() {
    const commonParams = {
      appId: route.query.appId as string,
      pageNum: 1,
      pageSize: 10,
    };

    const confList = await fetchSparkConfList(commonParams);
    const backupList = await fetchSparkBackUps(commonParams);
    const optionList = await fetchSparkOptionLog(commonParams);

    if (confList.records.length > 0) detailTabs.showConf = true;
    if (backupList.records.length > 0) detailTabs.showBackup = true;
    if (optionList.records.length > 0) detailTabs.showOptionLog = true;
  }

  /* Get yarn data */
  async function handleYarn() {
    yarn.value = await fetchSparkYarn();
  }

  // async function getExternalLinks() {
  //   const { data: links } = await fetchAppExternalLink({ appId: route.query.appId as string });
  //   externalLinks.value = links.data;
  // }

  onMounted(() => {
    // getExternalLinks();
  });

  onUnmounted(() => {
    pause();
  });

  const appNotRunning = computed(() => app.state !== AppStateEnum.RUNNING || yarn.value === null);
</script>
<template>
  <PageWrapper content-full-height content-background contentClass="p-24px">
    <div class="mb-15px">
      <span class="app-bar">{{ t('spark.app.detail.detailTitle') }}</span>
      <Space class="-mt-8px">
        <div v-for="link in externalLinks" :key="link.id">
          <LinkBadge
            :label="link.badgeLabel"
            :redirect="link.renderedLinkUrl"
            :color="link.badgeColor"
            :message="link.badgeName"
            :disabled="appNotRunning"
          />
        </div>
      </Space>
      <a-button type="primary" shape="circle" @click="router.back()" class="float-right -mt-8px">
        <Icon icon="ant-design:arrow-left-outlined" />
      </a-button>
      <a-button
        type="primary"
        @click="handleFlinkView"
        :disabled="appNotRunning"
        class="float-right -mt-8px mr-20px"
      >
        <Icon icon="ant-design:cloud-outlined" />
        {{ t('spark.app.detail.webUI') }}
      </a-button>
    </div>
    <Description @register="registerDescription" />
    <Divider class="mt-20px -mb-17px" />
    <DetailTab :app="app" :tabConf="detailTabs" />
    <Mergely @register="registerConfDrawer" :readOnly="true" />
    <!-- <RequestModal @register="registerOpenApi" /> -->
  </PageWrapper>
</template>
<style lang="less">
  @import url('../../flink/app/styles/Detail.less');
</style>
