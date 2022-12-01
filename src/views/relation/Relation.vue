<template>
  <div class="fill-height">
    <v-container v-if="web3 && connected" class="fill-height">
      <v-row justify="center">
        <v-col md="6">
          <v-card class="fill-width">
            <v-card outlined>
              <v-card-title>
                <v-avatar size="24" class="mr-2">
                  <img :src="require('@/assets/logo.png')" alt="DAO" />
                </v-avatar>
                <span class="title font-weight-light">
                  {{ $t("Query Relation") }}
                </span>
              </v-card-title>
              <v-divider></v-divider>
              <template
                v-if="(chainId === 56 || chainId === 128) && contractAddress"
              >
                <v-card-text>
                  <form>
                    <v-card-title>
                      <span class="headline">{{
                        $t("Please enter your address")
                      }}</span>
                    </v-card-title>
                    <v-card-text>
                      <v-text-field
                        :label="$t('Address')"
                        v-model="queryAccount"
                        :error-messages="queryAccountErrors"
                        required
                        @input="$v.queryAccount.$touch()"
                        @blur="$v.queryAccount.$touch()"
                        :autofocus="queryAccountFocus"
                        append-icon="mdi-close"
                        @click:append="appendIconCallback"
                      ></v-text-field>
                    </v-card-text>
                    <v-card-actions class="justify-center custom-btn">
                      <v-btn
                        large
                        color="#93B954"
                        dark
                        width="80%"
                        :disabled="!submitLoading"
                        @click="queryInvitee"
                      >
                        {{ $t("Query Invitee") }}
                      </v-btn>
                    </v-card-actions>
                    <v-card-actions class="justify-center custom-btn">
                      <v-btn
                        large
                        width="80%"
                        :disabled="!submitLoading"
                        @click="queryInviter"
                      >
                        {{ $t("Query Inviter") }}
                      </v-btn>
                    </v-card-actions>
                  </form>
                </v-card-text>
                <v-divider v-if="isQuery && dataList.length > 0"></v-divider>
                <v-list v-if="isQuery && dataList.length > 0">
                  <v-list-item
                    >{{ $t("Result Count") }}:
                    {{ dataList.length }}</v-list-item
                  >
                  <p class="ps-5 pe-5" v-for="item in dataList" :key="item">
                    {{ item }}
                    <!-- <v-divider
                      v-if="index < dataList.length - 1"
                      :key="index"
                    ></v-divider> -->
                  </p>
                  <!-- <v-list-item-group
                    v-for="(item, index) in dataList"
                    :key="item"
                    active-class="green--text"
                  >
                    <v-list-item
                      class="text-caption"
                      @click="handleCopy(item, $event)"
                    >
                      {{ item }}
                    </v-list-item>
                    <v-divider
                      v-if="index < dataList.length - 1"
                      :key="index"
                    ></v-divider>
                  </v-list-item-group> -->
                </v-list>
                <v-alert
                  v-else-if="isQuery && dataList.length <= 0"
                  class="ma-4"
                  outlined
                  type="warning"
                  prominent
                  border="left"
                >
                  {{ $t("No Data") }}
                </v-alert>
              </template>
              <template v-else>
                <v-card-text>
                  <v-row align="center">
                    <v-col class="subtitle-1" cols="12">
                      {{ $t("The current chain does not support operation") }}
                    </v-col>
                  </v-row>
                </v-card-text>
              </template>
            </v-card>
          </v-card>
          <v-card justify="center" class="fill-width mt-10">
            <v-card-title>
              <span class="title font-weight-light">
                {{ $t("Current Token Address") }}
              </span>
            </v-card-title>
            <v-divider></v-divider>
            <v-card-text>
              <v-row align="center">
                <v-col
                  class="body-1"
                  cols="12"
                  @click="handleCopy(address, $event)"
                >
                  <p>
                    {{ address }}
                    <v-icon>mdi-content-copy</v-icon>
                  </p>
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
          <v-overlay z-index="9999" opacity="0.7" :value="loading">
            <v-progress-circular indeterminate size="64"></v-progress-circular>
          </v-overlay>
          <!-- 提示层 -->
          <v-snackbar
            v-model="operationResult.snackbar"
            :color="operationResult.color"
            centered
            top
            timeout="4000"
          >
            {{ $t(operationResult.text) }}
          </v-snackbar>
        </v-col>
      </v-row>
    </v-container>
    <v-container v-else>
      <v-row justify="center">
        <v-col md="6">
          <v-card justify="center" class="fill-width">
            <v-card-actions class="justify-center">
              <!-- 连接钱包 -->
              <v-btn
                class="mr-2"
                v-if="!connected"
                color="#93B954"
                block
                @click="onConnect"
              >
                {{ $t("Connect Wallet") }}
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import { validationMixin } from "vuelidate";
import { required } from "vuelidate/lib/validators";
import clip from "@/utils/clipboard";
import { ZeroAddress, RelationContractAddress } from "@/constants";
import { getContractByABI, toChecksumAddress } from "@/utils/web3";
// 引入合约 ABI 文件
import RelationABI from "@/constants/abi/Relationship_abi.json";

export default {
  name: "Relation",
  mixins: [validationMixin],
  validations: {
    queryAccount: { required }
  },
  data: () => ({
    contractAddress: null,
    loading: false,
    queryAccountFocus: true,
    queryAccount: undefined,
    // 查询数据
    selected: [2],
    isQuery: false,
    dataList: [],
    // 提示框
    operationResult: {
      color: "success",
      snackbar: false,
      text: `Hello`
    }
  }),
  created() {
    if (!this.web3 || !this.connected) {
      this.onConnect();
    }
    if (this.chainId === 56 || this.chainId === 128) {
      this.contractAddress = RelationContractAddress[this.chainId];
    }
  },
  watch: {
    chainId(chainId) {
      this.contractAddress = RelationContractAddress[chainId];
    }
  },
  computed: {
    connected() {
      return this.$store.state.web3.connected;
    },
    web3() {
      return this.$store.state.web3.web3;
    },
    address() {
      return this.$store.state.web3.address;
    },
    chainId() {
      return this.$store.state.web3.chainId;
    },
    queryAccountErrors() {
      const errors = [];
      if (!this.$v.queryAccount.$dirty) return errors;
      !this.$v.queryAccount.required &&
        errors.push(this.$t("Please enter your address"));

      return errors;
    },
    submitLoading() {
      return this.queryAccount && this.queryAccountErrors.length <= 0;
    }
  },
  methods: {
    // 连接钱包 OK
    onConnect() {
      this.$store.dispatch("web3/connect");
    },
    // 断开连接钱包 OK
    closeWallet() {
      this.$store.dispatch("web3/closeWallet");
    },
    // 复制地址
    handleCopy(text, event) {
      clip(text, event);
      this.operationResult.color = "success";
      this.operationResult.snackbar = true;
      this.operationResult.text = "Cope Success";
    },
    appendIconCallback() {
      this.queryAccount = "";
      this.isQuery = false;
    },
    // 查询被邀请人
    async queryInvitee() {
      if (this.$v.$invalid) {
        // error info
        if (this.$v.queryAccount.$invalid) {
          this.queryAccountFocus = true;
        }
        this.$v.$touch();
      } else {
        this.$v.$touch();
        const contract = getContractByABI(
          RelationABI,
          this.contractAddress,
          this.web3
        );
        this.isQuery = false;
        this.loading = true;
        this.dataList = [];
        // 执行合约
        contract.methods
          .getMemberListByInviter(toChecksumAddress(this.queryAccount))
          .call({ from: this.address }, (e, r) => {
            if (e) {
              this.operationResult.color = "red";
              this.operationResult.snackbar = true;
              this.operationResult.text =
                "The current account does not have query permission";
              this.loading = false;
            } else {
              this.dataList = r;
              this.isQuery = true;
              this.loading = false;
            }
          });
      }
    },
    // 查询邀请人
    async queryInviter() {
      if (this.$v.$invalid) {
        // error info
        if (this.$v.queryAccount.$invalid) {
          this.queryAccountFocus = true;
        }
        this.$v.$touch();
      } else {
        this.$v.$touch();
        const contract = getContractByABI(
          RelationABI,
          this.contractAddress,
          this.web3
        );
        this.isQuery = false;
        this.loading = true;
        this.dataList = [];
        console.info(this.contractAddress);
        console.info(this.address);
        // 执行合约
        contract.methods
          .getInviterInfoByInvitee(toChecksumAddress(this.queryAccount))
          .call({ from: this.address }, (e, r) => {
            if (e) {
              this.operationResult.color = "red";
              this.operationResult.snackbar = true;
              this.operationResult.text =
                "The current account does not have query permission";
              this.loading = false;
            } else {
              if (r.inviterToken != ZeroAddress) {
                this.dataList.push(r.inviterToken);
              }
              this.isQuery = true;
              this.loading = false;
            }
          });
      }
    }
  }
};
</script>

<style lang="sass">
.custom-btn
  .theme--dark.v-btn.v-btn--disabled.v-btn--has-bg
    background-color: rgb(147, 185, 84) !important
    border-color: rgb(147, 185, 84) !important
    opacity: 0.5 !important

  .v-btn--disabled
    background-color: rgb(147, 185, 84)
    border-color: rgb(147, 185, 84)
    opacity: 0.5
</style>
