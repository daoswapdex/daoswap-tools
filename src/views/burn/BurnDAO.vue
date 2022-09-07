<template>
  <div class="fill-height">
    <v-container v-if="web3 && connected" class="fill-height">
      <v-row justify="center">
        <v-col md="6">
          <!-- 操作 -->
          <v-card class="fill-width mt-10">
            <v-card outlined>
              <v-card-title>
                <v-avatar size="24" class="mr-2">
                  <img :src="require('@/assets/logo.png')" alt="DAO" />
                </v-avatar>
                <span class="title font-weight-bold text-h5">
                  {{ $t("Burn") }}
                </span>
              </v-card-title>
              <v-divider></v-divider>
              <v-card-text v-if="isOpen">
                <v-card-text>
                  <v-row align="center">
                    <v-col class="subtitle-1" cols="12">
                      DAO {{ $t("Token.Address") }}:
                      {{ DAOAddress }}
                    </v-col>
                    <v-col class="subtitle-1" cols="12">
                      DAO {{ $t("Token.Symbol") }}:
                      {{ tokenSymbol }}
                    </v-col>
                    <v-col class="subtitle-1" cols="12">
                      DAO {{ $t("Token.TotalSupply") }}:
                      {{ totalSupply }}
                    </v-col>
                    <v-col class="subtitle-1" cols="12">
                      DAO {{ $t("Token.Balance") }}:
                      {{ accountAssets.balance }}
                    </v-col>
                  </v-row>
                </v-card-text>
                <v-divider></v-divider>
                <form>
                  <v-card-text>
                    <v-text-field
                      :label="
                        `${$t(
                          'At least'
                        )} ${minBurnAmount} ${tokenSymbol}, ${$t(
                          'At mostest'
                        )} ${maxBurnAmount} ${tokenSymbol}`
                      "
                      v-model="burnAmount"
                      :error-messages="burnAmountErrors"
                      required
                      @input="$v.burnAmount.$touch()"
                      @blur="$v.burnAmount.$touch()"
                      :autofocus="burnAmountFocus"
                    ></v-text-field>
                  </v-card-text>
                  <v-card-actions class="justify-center">
                    <v-btn
                      large
                      color="#93B954"
                      dark
                      width="80%"
                      :disabled="
                        !(
                          submitLoading &&
                          accountAssets.balance > 0 &&
                          parseFloat(accountAssets.balance) >=
                            parseFloat(burnAmount)
                        )
                      "
                      @click="submit()"
                    >
                      {{ $t("Burn") }}
                    </v-btn>
                  </v-card-actions>
                </form>
              </v-card-text>
              <v-card-text v-else>
                <v-row align="center">
                  <v-col class="subtitle-1" cols="12">
                    {{ $t("The current chain does not support operation") }}
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </v-card>
          <!-- 当前钱包账号 -->
          <v-card justify="center" class="fill-width mt-10">
            <v-card-title>
              <span class="title font-weight-bold text-h5">
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
          <!-- 官方说明 -->
          <v-card justify="center" class="fill-width mt-10">
            <v-card-text>
              <v-row align="center">
                <v-col class="body-1" cols="12">
                  <p @click="handleCopy(DAOAddress, $event)">
                    DAO contract: {{ DAOAddress }}
                    <v-icon>mdi-content-copy</v-icon>
                  </p>
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
          <!-- 遮罩层 -->
          <v-overlay z-index="9999" opacity="0.7" :value="loading">
            <v-progress-circular indeterminate size="64"></v-progress-circular>
          </v-overlay>
          <!-- 提示层 -->
          <v-snackbar
            v-model="operationResult.snackbar"
            :color="operationResult.color"
            centered
            top
            timeout="5000"
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
import { required, decimal } from "vuelidate/lib/validators";
import clip from "@/utils/clipboard";
import {
  DAOContractAddress,
  WHITE_LISTS_SWITCH,
  WHITE_LISTS
} from "@/constants";
import { getContractByABI, weiToEther, etherToWei } from "@/utils/web3";
// 引入合约 ABI 文件
import ERC20DAO from "@/constants/abi/ERC20_abi.json";

export default {
  name: "BurnDAO",
  mixins: [validationMixin],
  validations: {
    burnAmount: { required, decimal }
  },
  data: () => ({
    loading: false,
    supplyChainIds: [56, 128, 256],
    DAOAddress: null,
    tokenSymbol: "DAO",
    totalSupply: 0,
    // 表单数据
    burnAmountFocus: true,
    burnAmount: undefined,
    // 当前账户相关信息
    accountAssets: {
      balance: 0
    },
    // 合约数据
    minBurnAmount: 1,
    maxBurnAmount: 0,
    isOpen: false,
    // 提示框
    operationResult: {
      color: "success",
      snackbar: false,
      text: `Hello`
    }
  }),
  created() {
    if (this.supplyChainIds.indexOf(this.chainId) > -1) {
      this.DAOAddress = DAOContractAddress[this.chainId];
    }
    if (this.web3 && this.connected) {
      this.getInfo();
    } else {
      this.onConnect();
    }
  },
  watch: {
    web3(web3) {
      if (web3) {
        this.getInfo();
      }
    },
    address(address) {
      if (address) {
        this.getInfo();
      }
    },
    chainId(chainId) {
      this.DAOAddress = DAOContractAddress[chainId];
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
      // return "0xCD4BBF4FB76d400Eab42B9e530BB98BC72fFC20E";
    },
    chainId() {
      return this.$store.state.web3.chainId;
    },
    burnAmountErrors() {
      const errors = [];
      if (!this.$v.burnAmount.$dirty) return errors;
      !this.$v.burnAmount.decimal &&
        errors.push(this.$t("BurnForm.Invalid amount"));
      !this.$v.burnAmount.required &&
        errors.push(this.$t("BurnForm.The amount is required"));

      const burnAmountValue = parseFloat(this.$v.burnAmount.$model);
      if (burnAmountValue <= 0) {
        errors.push(this.$t("BurnForm.The amount is be gt zero"));
      }
      if (!this.isWhitelist) {
        if (burnAmountValue > 0 && burnAmountValue < this.minBurnAmount) {
          errors.push(
            this.$t("BurnForm.The amount does not meet the requirements")
          );
        }
        if (parseFloat(burnAmountValue) > this.maxBurnAmount) {
          errors.push(
            this.$t("BurnForm.The amount exceeds the max burn amount")
          );
        }
      }
      if (burnAmountValue > this.accountAssets.balance) {
        errors.push(this.$t("BurnForm.The amount exceeds the balance"));
      }
      return errors;
    },
    submitLoading() {
      return this.burnAmount && this.burnAmountErrors.length <= 0;
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
    // 获取信息
    async getInfo() {
      this.loading = true;
      try {
        await this.getContractInfo();
      } catch (error) {
        console.info(error);
      }
      this.loading = false;
    },
    // 获取合约信息
    async getContractInfo() {
      try {
        this.DAOAddress = DAOContractAddress[this.chainId];
        // 查询当前账号余额
        const contract = getContractByABI(ERC20DAO, this.DAOAddress, this.web3);
        const balance = await contract.methods.balanceOf(this.address).call();
        this.accountAssets.balance = weiToEther(balance, this.web3);
        this.tokenSymbol = await contract.methods.symbol().call();
        const totalSupply = await contract.methods.totalSupply().call();
        this.totalSupply = weiToEther(totalSupply, this.web3);

        this.maxBurnAmount = this.accountAssets.balance;
        this.isOpen =
          this.supplyChainIds.indexOf(this.chainId) > -1 &&
          WHITE_LISTS_SWITCH &&
          WHITE_LISTS.indexOf(this.address) > -1;
      } catch (error) {
        console.info(error);
      }
    },
    // 提交
    async submit() {
      if (this.$v.$invalid) {
        // error info
        if (this.$v.burnAmount.$invalid) {
          this.burnAmountFocus = true;
        }
        this.$v.$touch();
      } else {
        this.$v.$touch();
        this.loading = true;
        if (
          parseFloat(this.burnAmount) <= parseFloat(this.accountAssets.balance)
        ) {
          const contract = getContractByABI(
            ERC20DAO,
            this.DAOAddress,
            this.web3
          );
          contract.methods
            .burn(etherToWei(this.burnAmount, this.web3))
            .send({ from: this.address })
            .then(() => {
              this.loading = false;
              this.operationResult.color = "success";
              this.operationResult.snackbar = true;
              this.operationResult.text = "Burn Success";
              this.burnAmount = null;
              this.getInfo();
            })
            .catch(e => {
              this.loading = false;
              console.info(e);
            });
        } else {
          this.operationResult.color = "error";
          this.operationResult.snackbar = true;
          this.operationResult.text =
            "BurnForm.The amount exceeds the allowance amount";
          this.loading = false;
        }
      }
    }
  }
};
</script>

<style lang="sass">
.theme--dark.v-btn.v-btn--disabled.v-btn--has-bg
  background-color: rgb(147, 185, 84) !important
  border-color: rgb(147, 185, 84) !important
  opacity: 0.5 !important

.v-btn--disabled
  background-color: rgb(147, 185, 84)
  border-color: rgb(147, 185, 84)
  opacity: 0.5
</style>
