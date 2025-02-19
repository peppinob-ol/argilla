<template>
  <HeaderAndTopAndOneColumn v-if="!$fetchState.pending && !$fetchState.error">
    <template v-slot:header>
      <HeaderFeedbackTaskComponent
        :key="datasetName && workspace"
        :datasetId="datasetId"
        :breadcrumbs="breadcrumbs"
        :showTrainButton="true"
        @on-click-train="showTrainModal(true)"
      />
      <BaseModal
        :modal-custom="true"
        :prevent-body-scroll="true"
        modal-class="modal-auto"
        modal-position="modal-top-center"
        :modal-visible="visibleTrainModal"
        allow-close
        @close-modal="showTrainModal(false)"
      >
        <DatasetTrainComponent
          datasetTask="FeedbackTask"
          :datasetName="datasetName"
          :workspaceName="workspace"
        />
      </BaseModal>
    </template>
    <template v-slot:sidebar-right>
      <SidebarFeedbackTaskComponent
        :datasetId="datasetId"
        @refresh="onRefresh()"
      />
    </template>
    <template v-slot:top>
      <DatasetFiltersComponent :datasetId="datasetId" />
    </template>
    <template v-slot:center>
      <CenterFeedbackTaskContent :datasetId="datasetId" />
    </template>
    <template v-slot:footer>
      <PaginationFeedbackTaskComponent :datasetId="datasetId" />
    </template>
  </HeaderAndTopAndOneColumn>
</template>

<script>
import HeaderAndTopAndOneColumn from "@/layouts/HeaderAndTopAndOneColumn";
import {
  RECORD_STATUS,
  deleteAllRecords,
} from "@/models/feedback-task-model/record/record.queries";
import { deleteAllRecordFields } from "@/models/feedback-task-model/record-field/recordField.queries";
import { deleteAllRecordResponses } from "@/models/feedback-task-model/record-response/recordResponse.queries";
import {
  upsertFeedbackDataset,
  getFeedbackDatasetNameById,
  getFeedbackDatasetWorkspaceNameById,
} from "@/models/feedback-task-model/feedback-dataset/feedbackDataset.queries";
import { LABEL_PROPERTIES } from "@/components/feedback-task/feedbackTask.properties";
import { Notification } from "@/models/Notifications";

const TYPE_OF_FEEDBACK = Object.freeze({
  ERROR_FETCHING_DATASET_INFO: "ERROR_FETCHING_DATASET_INFO",
  ERROR_FETCHING_WORKSPACE_INFO: "ERROR_FETCHING_WORKSPACE_INFO",
});

export default {
  name: "DatasetPage",
  components: {
    HeaderAndTopAndOneColumn,
  },
  data() {
    return {
      areResponsesUntouched: true, // NOTE - this flag is used to show or to not show a toast when questionnaire is touched (to prevent loosing current modification)
      visibleTrainModal: false, // TODO - encapsulate this logic in datasetTrain.component and create new datasetTrain.modal
    };
  },
  beforeRouteLeave(to, from, next) {
    const isNotificationForThisRoute =
      !this.areResponsesUntouched &&
      ["datasets", "dataset-id-settings", "user-settings"].includes(to.name);

    if (isNotificationForThisRoute) {
      this.showNotification({
        eventToFireOnClick: next,
        message: this.toastMessageOnLeavingRoute,
        buttonMessage: this.buttonMessage,
      });
    } else {
      next();
    }
  },
  computed: {
    datasetId() {
      return this.$route.params.id;
    },
    datasetName() {
      return getFeedbackDatasetNameById(this.datasetId);
    },
    workspace() {
      return getFeedbackDatasetWorkspaceNameById(this.datasetId);
    },
    breadcrumbs() {
      return [
        { link: { name: "datasets" }, name: "Home" },
        {
          link: { path: `/datasets?workspaces=${this.workspace}` },
          name: this.workspace,
        },
        {
          link: {
            name: null,
            params: { workspace: this.workspace, dataset: this.datasetName },
          },
          name: this.datasetName,
        },
      ];
    },
  },
  async fetch() {
    try {
      // 1- fetch dataset info
      const dataset = await this.getDatasetInfo(this.datasetId);

      // TODO - remove step 2 when workspace name will be include in the getDatasetInfo API call
      // 2- fetch workspace info
      const workspace = await this.getWorkspaceInfo(dataset.workspace_id);

      // 3- insert in ORM
      upsertFeedbackDataset({ ...dataset, workspace_name: workspace });

      // Check if response is untouched
      this.onBusEventAreResponsesUntouched();
    } catch (err) {
      this.manageErrorIfFetchNotWorking(err);
    }
  },
  created() {
    this.checkIfUrlHaveRecordStatusOrInitiateQueryParams();

    this.toastMessageOnRefresh =
      "Your changes will be lost if you refresh the page";
    this.toastMessageOnLeavingRoute =
      "Your changes will be lost if you leave the current page";
    this.buttonMessage = LABEL_PROPERTIES.CONTINUE;
    this.typeOfToast = "warning";
  },
  methods: {
    checkIfUrlHaveRecordStatusOrInitiateQueryParams() {
      this.$route.query?._status ??
        this.$router.replace({
          query: {
            ...this.$route.query,
            _search: "",
            _page: 1,
            _status: RECORD_STATUS.PENDING.toLowerCase(),
          },
        });
    },
    async getDatasetInfo(datasetId) {
      try {
        const { data } = await this.$axios.get(`/v1/datasets/${datasetId}`);

        return data;
      } catch (err) {
        throw {
          response: TYPE_OF_FEEDBACK.ERROR_FETCHING_DATASET_INFO,
        };
      }
    },
    async getWorkspaceInfo(workspaceId) {
      try {
        const { data: responseWorkspace } = await this.$axios.get(
          `/v1/workspaces/${workspaceId}`
        );

        const { name } = responseWorkspace || { name: null };

        return name;
      } catch (err) {
        throw {
          response: TYPE_OF_FEEDBACK.ERROR_FETCHING_WORKSPACE_INFO,
        };
      }
    },
    manageErrorIfFetchNotWorking({ response }) {
      this.initErrorNotification(response);
      this.$router.push("/");
    },
    initErrorNotification(response) {
      let message = "";
      switch (response) {
        case TYPE_OF_FEEDBACK.ERROR_FETCHING_DATASET_INFO:
          message = `Can't get dataset info for dataset_id: ${this.datasetId}`;
          break;
        case TYPE_OF_FEEDBACK.ERROR_FETCHING_WORKSPACE_INFO:
          message = `Can't get workspace info for dataset_id: ${this.datasetId}`;
          break;
        default:
          message = `There was an error on fetching dataset info and workspace info. Please try again`;
      }

      const paramsForNotitification = {
        message,
        numberOfChars: message.length,
        type: "error",
      };

      Notification.dispatch("notify", paramsForNotitification);
    },
    async onRefresh() {
      if (this.areResponsesUntouched) {
        await this.deleteRecordsAndRefreshDataset();
      } else {
        this.showNotification({
          eventToFireOnClick: async () => {
            await this.deleteRecordsAndRefreshDataset();
          },
          message: this.toastMessageOnRefresh,
          buttonMessage: this.buttonMessage,
          typeOfToast: "warning",
        });
      }
    },
    async deleteRecordsAndRefreshDataset() {
      await deleteAllRecords();
      await deleteAllRecordFields();
      await deleteAllRecordResponses();
      this.$fetch();
    },
    async onBusEventAreResponsesUntouched() {
      this.$root.$on("are-responses-untouched", (areResponsesUntouched) => {
        this.areResponsesUntouched = areResponsesUntouched;
      });
    },
    showNotification({
      eventToFireOnClick,
      message,
      buttonMessage,
      typeOfToast,
    }) {
      Notification.dispatch("notify", {
        message: message ?? "",
        numberOfChars: 500,
        type: typeOfToast ?? "warning",
        buttonText: buttonMessage ?? "",
        async onClick() {
          eventToFireOnClick();
        },
      });
    },
    showTrainModal(value) {
      this.visibleTrainModal = value;
    },
  },
  destroyed() {
    this.$root.$off("are-responses-untouched");
    Notification.dispatch("clear");
  },
};
</script>
