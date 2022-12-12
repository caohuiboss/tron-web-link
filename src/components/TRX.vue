<template>
  <!-- <a-spin :spinning="fullscreenLoading"> -->
  <div class="card-header">
    <span class="title">BatchSender</span>
    <a-space>
      <a-button
        v-if="!ownerAddress"
        type="primary"
        shape="round"
        class="base-button"
        @click="linkWallet"
        >链接钱包</a-button
      >
      <a-tooltip placement="right">
        <template #title>
          每转一个地址就给接收地址增加一点积分，每个地址转账需要一积分
        </template>
        <a-space v-if="ownerAddress" class="radius-box">
          <div class="ownerAddress" v-if="ownerAddress">
            {{ uzipAddress(ownerAddress) }}
          </div>
          <span class="rate">10086 积分</span>
          <DollarOutlined class="shake" />
        </a-space>
      </a-tooltip>
    </a-space>
  </div>
  <div class="page-wrapper">
    <div class="form-wrapper">
      <a-form layout="vertical" ref="form" :model="formData">
        <a-form-item label="转账币种">
          <a-radio-group v-model:value="formData.currency">
            <a-radio :value="0">TRX</a-radio>
            <a-radio :value="1">代币</a-radio>
          </a-radio-group>
        </a-form-item>
        <a-form-item label="代币合约" v-if="formData.currency === 1">
          <a-input
            placeholder="请输入合法的代币地址"
            v-model:value="formData.contractAddress"
            @blur="contractAddressBlur"
          ></a-input>
        </a-form-item>
        <a-form-item v-if="formData.currency === 1">
          <template #label>
            <a-space>
              <span>代币精度</span>
              <a-tooltip placement="right">
                <template #title>
                  精度请填写 1-18 可保证代币转账数量与填写的数量一致
                  我们会自动读取代币合约的ABI获取精度 如果没有ABI
                  需要你手动确认填写(精度可在钱包查看代币详情获取)
                </template>
                <QuestionCircleFilled class="shake" />
              </a-tooltip>
            </a-space>
          </template>
          <a-input-number
            placeholder="请输入代币精度"
            :min="1"
            :max="18"
            style="width: 100%"
            :precision="0"
            v-model:value="formData.precision"
          />
        </a-form-item>
        <a-form-item label="转账方式">
          <a-radio-group v-model:value="formData.way">
            <!-- <a-radio :value="0">单个转账</a-radio> -->
            <a-radio :value="1">批量转账</a-radio>
            <a-radio :value="2">不同数量批量转账</a-radio>
          </a-radio-group>
        </a-form-item>
        <a-form-item label="下载模板" v-if="formData.way === 2">
          <a
            type="link"
            href="/txt/template.txt"
            download="不同数量批量转账模板.txt"
            target="_blank"
            >不同数量批量转账模板</a
          >
        </a-form-item>
        <a-form-item label="转账数量" v-if="formData.way !== 2">
          <a-input-number
            v-model:value="formData.num"
            style="width: 100%"
            :min="0"
          ></a-input-number>
        </a-form-item>
        <a-form-item v-if="formData.way === 0" label="收款人地址(单个地址)">
          <a-input v-model:value="formData.receiveAddress"></a-input>
        </a-form-item>
        <a-form-item
          v-else-if="formData.way === 1"
          label="收款人地址(多个地址换行)"
        >
          <codemirror
            v-model="formData.receiveAddress"
            placeholder="仅仅只需要多个地址换行"
            :style="{ height: '300px' }"
            :autofocus="true"
            :extensions="[oneDark]"
            :indent-with-tab="true"
            :tab-size="2"
          />
        </a-form-item>
        <a-form-item v-else label="收款人地址(多个地址换行)">
          <codemirror
            v-model="formData.receiveAddress"
            placeholder="地址|数量  多个换行 可以从模板中复制过来"
            :style="{ height: '300px' }"
            :autofocus="true"
            :extensions="[oneDark]"
            :indent-with-tab="true"
            :tab-size="2"
          />
        </a-form-item>
        <a-form-item label="备注">
          <a-input
            v-model:value="formData.remark"
            placeholder="请输入备注"
          ></a-input>
        </a-form-item>
        <a-form-item style="text-align: center">
          <div class="button-wrapper">
            <a-button
              v-show="isResult || ownerAddress"
              type="primary"
              size="large"
              class="base-button button-send"
              @click="onSubmit"
              :disabled="!ownerAddress"
              :loading="fullscreenLoading"
              >发送</a-button
            >
            <!-- <a-button class="button-send" @click="reload">刷新页面</a-button> -->
          </div>
        </a-form-item>
        <!-- <a-form-item v-if="tableData.length > 0">
            <a-table size="small" :columns="columns" :data-source="tableData">
            </a-table>
          </a-form-item> -->
      </a-form>
    </div>
  </div>
</template>

<script setup lang="jsx">
import { notification, message } from "ant-design-vue";
import { QuestionCircleFilled, DollarOutlined } from "@ant-design/icons-vue";
import { onMounted, reactive, ref, watch, h } from "vue";
import { Codemirror } from "vue-codemirror";
import { oneDark } from "@codemirror/theme-one-dark";

const CONTANS = 1000000;
const ownerAddress = ref();
const isResult = ref(false);
const tronWeb = ref(null);
const fullscreenLoading = ref(false);
const tableData = ref([
  {
    receiveAddress: "地址",
    num: "数量",
    result: "结果",
  },
]);

const formData = reactive({
  contractAddress: undefined,
  way: 1,
  num: 0.1,
  currency: 0,
  remark: "",
  precision: 18,
  receiveAddress: undefined,
});

// 提交
const onSubmit = async () => {
  if (!formData.receiveAddress) {
    message.warning("请填写转账地址");
    return;
  }
  if (
    formData.currency === 1 &&
    !tronWeb.value.isAddress(formData.contractAddress)
  ) {
    message.error("请填写正确的合约地址");
    return;
  }
  const hide = message.loading("正在发送代币中 请耐心等待", 0);
  try {
    isResult.value = false;
    fullscreenLoading.value = true;
    if (formData.way === 0) {
      await transaction(formData.receiveAddress);
    } else if (formData.way === 1) {
      const receiveAddress = dealAddresses(formData.receiveAddress) || [];
      for (let index = 0; index < receiveAddress.length; index++) {
        const address = receiveAddress[index];
        if (!tronWeb.value.isAddress(address)) {
          message.error("请填写合法接收地址");
          return;
        }
      }
      if (receiveAddress.includes(ownerAddress.value)) {
        message.error("不能给自己转账");
        return;
      }
      for (let index = 0; index < receiveAddress.length; index++) {
        if (formData.currency === 0) {
          await transaction(receiveAddress[index]);
        } else {
          await transactionToken(receiveAddress[index], formData.num);
        }
      }
    } else {
      const receiveAddress = dealAddresses(formData.receiveAddress);
      for (let index = 0; index < receiveAddress.length; index++) {
        const splitAddress = receiveAddress[index].split("|");
        if (splitAddress.length === 2) {
          if (formData.currency === 1) {
            await transactionToken(splitAddress[0], +splitAddress[1]);
          } else {
            await transaction(splitAddress[0], +splitAddress[1]);
          }
        }
      }
    }
    notification.open({
      message: "发币结果",
      description: () =>
        tableData.value.map((item, i) => {
          return h(
            "div",
            {
              class: "result-title",
            },
            [
              h(
                "div",
                { class: "result-item" },
                i === 0 ? item.receiveAddress : uzipAddress(item.receiveAddress)
              ),
              h("div", { class: "result-item" }, item.num),
              h("div", { class: "result-item" }, item.result),
            ]
          );
        }),
      duration: 0,
    });
  } catch (error) {
    message.error(error);
  } finally {
    isResult.value = true;
    fullscreenLoading.value = false;
    hide();
  }
};

// 链接钱包
const linkWallet = () => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
    ownerAddress.value = tronWeb.value.defaultAddress.base58;
  } else {
    message.warning("请下载波宝钱包浏览器插件");
  }
};

// 查询合约
const contractAddressBlur = async () => {
  if (formData.contractAddress) {
    if (
      formData.currency === 1 &&
      !tronWeb.value.isAddress(formData.contractAddress)
    ) {
      message.error("请填写正确的合约地址");
      return;
    }
    const res = await tronWeb.value.trx.getContract(formData.contractAddress);
    if (res?.abi) {
      const contract = await tronWeb.value.contract(
        res.abi.entrys,
        formData.contractAddress
      );
      if (contract.decimals) {
        const decimals = await contract.decimals().call();
        formData.precision = Number.isFinite(+decimals)
          ? +decimals
          : +tronWeb.value.toDecimal(decimals._hex) || 18;
      } else {
        message.error("该代币未暴露获取精度方法 精度需你手动确认填写");
        formData.precision = 18;
      }
    } else {
      message.error("未查询到该代币的ABI 精度需你手动确认填写");
    }
  }
};

// 处理地址
const dealAddresses = (str) =>
  str
    .replace(/(\r\n)|(\n)/g, ",")
    .split(",")
    .filter((v) => v && v.trim());

// trx转账交易
const transactionTrx = async (receiveAddress, num = 1) => {
  const tx = await tronWeb.value.transactionBuilder.sendTrx(
    receiveAddress,
    num * CONTANS,
    ownerAddress.value
  );
  const unsignedTransaction =
    await tronWeb.value.transactionBuilder.addUpdateData(
      tx,
      formData.remark,
      "utf-8"
    );
  const signedTx = await tronWeb.value.trx.sign(unsignedTransaction);
  const broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  console.log("TRX发放结果: ", receiveAddress);
  console.log(broastTx);
  tableData.value.push({
    receiveAddress,
    num: num || formData.num,
    result: broastTx.result ? "成功" : "失败",
  });
  const a = "background: #606060; color: #fff;";
  const b = "background: #1475B2; color: #fff;";
  console.log(
    ` %c 地址: ${receiveAddress} %c 数量: ${num || formData.num} %c 结果: ${
      broastTx.result || broastTx.code
    } `,
    a,
    b,
    a
  );
};

/**
 * 代币转账
 * @param {String} receiveAddress 需要接收的地址
 * @param {Number} num 需要转账的数量
 */
const transactionToken = async (receiveAddress, num) => {
  const parameter = [
    { type: "address", value: receiveAddress },
    {
      type: "uint256",
      value: tronWeb.value
        .toBigNumber(num * 10 ** formData.precision)
        .toString(10),
    },
  ];
  const tx = await tronWeb.value.transactionBuilder.triggerSmartContract(
    formData.contractAddress,
    "transfer(address,uint256)",
    {},
    parameter,
    ownerAddress.value
  );
  await tronWeb.value.transactionBuilder.addUpdateData(
    tx.transaction,
    formData.remark,
    "utf-8"
  );
  const signedTx = await tronWeb.value.trx.sign(tx.transaction);
  const broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  const contract = await tronWeb.value.contract().at(formData.contractAddress);
  await contract.decimals().call();

  const a = "background: #606060; color: #fff;";
  const b = "background: #1475B2; color: #fff;";
  tableData.value.push({
    receiveAddress,
    num: num || formData.num,
    result: broastTx.result ? "成功" : "失败",
  });
  console.log(
    ` %c 地址: ${receiveAddress} %c 数量: ${num || formData.num} %c 结果: ${
      broastTx.result || broastTx.code
    } `,
    a,
    b,
    a
  );
};

// 判断转代币还是TRX
const transaction = async (receiveAddress) => {
  if (formData.currency === 0) {
    await transactionTrx(receiveAddress, formData.num);
  } else {
    await transactionToken(receiveAddress, formData.num);
  }
};

// 切割地址
const uzipAddress = (str) => {
  if (!str) return "-";
  return str.substr(0, 8) + "..." + str.substr(-4, 4);
};

// 清空表单数据
const clearFormData = () => {
  formData.receiveAddress = undefined;
};

// 刷新页面
const reload = () => {
  window.location.reload();
};

// 监听表单数据变化
watch(
  () => [formData.currency, formData.way],
  () => {
    clearFormData();
  }
);

// 初始化
onMounted(() => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
  }
  window.addEventListener("message", (e) => {
    if (e.data.message && e.data.message.action == "accountsChanged") {
      linkWallet();
    }
  });
});
</script>

<style>
.ͼ1 .cm-scroller,
.cm-placeholder {
  font-family: Consolas, Monaco, "Andale Mono", "Ubuntu Mono", monospace;
}

.ͼ1 .cm-line {
  display: block;
  padding: 0 2px 0 6px;
}

.page-wrapper {
  width: 900px;
  margin: 20px auto;
  padding: 20px;
  border: 1px solid #e8e8e8;
  border-radius: 10px;
  background: rgba(36, 50, 100, 0.6);
  backdrop-filter: blur(3px);
  border-image-slice: 1;
  border-image-source: linear-gradient(
    92.7deg,
    #a310fe 10.86%,
    #19ee48 101.11%
  );
}

@media screen and (max-width: 768px) {
  .page-wrapper {
    width: 96%;
    padding: 20px;
    margin: 20px auto;
  }
}

.base-button {
  background: linear-gradient(
    90.87deg,
    #a310fe -41.78%,
    #b753f5 100%
  ) !important;
  /* box-shadow: 0px 3px 12px rgba(83, 100, 172, 0.5) !important; */
  border-radius: 10px !important;
  color: white !important;
}

.button-send {
  width: 55%;
}

.ant-form-item-label {
  font-weight: bold;
}

.ant-form-item-label > label {
  position: relative;
  display: inline-flex;
  align-items: center;
  max-width: 100%;
  height: 32px;
  color: rgba(253, 253, 253, 1) !important;
  font-size: 14px;
}

.ant-radio-group span {
  color: rgba(253, 253, 253, 1) !important;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  position: sticky;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 12px;
  min-height: 52px;
  position: sticky;
  top: 0;
  z-index: 20;
}

.loading-text {
  font-size: 20px;
  font-weight: bold;
  color: #b753f5;
}

.ant-spin-nested-loading > div > .ant-spin {
  min-height: 100vh !important;
}

.card-header > .title {
  font-size: 16px;
  font-weight: bold;
}

.radius-box {
  border-radius: 20px;
  border: 1px solid rgb(219, 219, 219);
  padding: 4px 8px 4px 4px;
}

.ownerAddress {
  border-radius: 20px;
  padding: 0 8px;
}

.ant-input,
.ant-input-number {
  border-radius: 8px !important;
  color: #e8e8e8 !important;
}

.button-wrapper {
  display: flex;
  justify-content: space-around;
  align-items: center;
}

.form-wrapper {
  margin: 10px auto;
}

.text {
  font-size: 14px;
}

.item {
  margin-bottom: 18px;
}

.box-card {
  width: 480px;
}

.result-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  border: 1px solid rgb(235, 235, 235);
  padding: 4px;
  border-radius: 2px;
}

.result-item {
  width: 30%;
  text-align: center;
}

.result-item:nth-child(1) {
  width: 40%;
}

.ml-2 {
  margin-left: 10px;
}

.shake {
  animation: shake 1000ms ease-in-out infinite;
}

.rate {
  font-size: 14px;
  color: #fcfcfc;
  font-weight: bold;
}

.ant-input {
  background-color: #2e3057 !important;
  height: 36px;
  border: 1px solid rgba(53, 201, 255, 0.2);
  color: rgba(255, 255, 255, 0.8);
}

.ant-input::placeholder {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.65);
}

.ant-input-number {
  width: 100%;
  background-color: #2e3057 !important;
  border: 1px solid rgba(53, 201, 255, 0.2);
  color: rgba(255, 255, 255, 0.8);
}

.ant-input-number-input {
  height: 36px;
}

.ant-input-number-input::placeholder {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.65);
}

@keyframes shake {
  /* 垂直抖动，核心代码 */
  10%,
  90% {
    transform: translate3d(0, -1px, 0);
  }

  20%,
  80% {
    transform: translate3d(0, +2px, 0);
  }

  30%,
  70% {
    transform: translate3d(0, -3px, 0);
  }

  40%,
  60% {
    transform: translate3d(0, +3px, 0);
  }

  50% {
    transform: translate3d(0, -3px, 0);
  }
}
</style>
