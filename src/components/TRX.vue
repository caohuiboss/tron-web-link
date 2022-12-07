<template>
  <a-spin :spinning="fullscreenLoading">
    <a-card>
      <template #title>
        <div class="card-header">
          <span>{{
            (ownerAddress && uzipAddress(ownerAddress)) || "请链接钱包"
          }}</span>
          <a-space>
            <a-tooltip placement="right">
              <template #title>
                每转一个地址就给接收地址增加一点积分，每个地址转账需要一积分
              </template>
              <a-space v-if="ownerAddress">
                <DollarOutlined class="shake" />
                <span class="rate">10086 积分</span>
              </a-space>
            </a-tooltip>

            <a-button v-if="!ownerAddress" type="primary" @click="linkWallet"
              >链接钱包</a-button
            >
          </a-space>
        </div>
      </template>
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
            <a-radio :value="0">单个转账</a-radio>
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
          <a-textarea
            v-model:value="formData.receiveAddress"
            placeholder="仅仅只需要多个地址换行"
            :autosize="{ minRows: 4 }"
          ></a-textarea>
        </a-form-item>
        <a-form-item v-else label="收款人地址(多个地址换行)">
          <a-textarea
            v-model:value="formData.receiveAddress"
            placeholder="地址|数量  多个换行 可以从模板中复制过来"
            :autosize="{ minRows: 4 }"
          ></a-textarea>
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
              v-show="isResult"
              type="primary"
              class="button-send"
              @click="onSubmit"
              :disabled="!ownerAddress"
              >发放</a-button
            >
            <a-button class="button-send" @click="reload">刷新页面</a-button>
          </div>
        </a-form-item>
        <a-form-item v-if="tableData.length > 0">
          <a-table size="small" :columns="columns" :data-source="tableData">
          </a-table>
        </a-form-item>
      </a-form>
    </a-card>
    <template #tip>
      <p>发放中，请耐心等待</p>
      <a-table bordered size="small" :columns="columns" :data-source="tableData">
      </a-table>
    </template>
  </a-spin>
</template>

<script setup>
import { notification, message } from "ant-design-vue";
import { QuestionCircleFilled, DollarOutlined } from "@ant-design/icons-vue";
import { onMounted, reactive, ref, watch } from "vue";

const CONTANS = 1000000;
const ownerAddress = ref();
const isResult = ref(true);
const tronWeb = ref(null);
const fullscreenLoading = ref(false);
const tableData = ref([]);

const formData = reactive({
  contractAddress: undefined,
  way: 1,
  num: 0.2,
  currency: 1,
  remark: "",
  precision: 18,
  receiveAddress: undefined,
});

const columns = ref([
  {
    title: "地址",
    align: "center",
    dataIndex: "receiveAddress",
    customRender: ({ record }) =>
      record.receiveAddress && uzipAddress(record.receiveAddress),
  },
  {
    title: "数量",
    align: "center",
    dataIndex: "num",
  },
  {
    title: "发送结果",
    align: "center",
    dataIndex: "result",
  },
]);

const linkWallet = () => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
    ownerAddress.value = tronWeb.value.defaultAddress.base58;
  } else {
    message.warning("请下载波宝钱包浏览器插件");
  }
};

const contractAddressBlur = async () => {
  if (formData.contractAddress) {
    if (
      formData.currency === 1 &&
      !tronWeb.value.isAddress(formData.contractAddress)
    ) {
      notification.error({
        message: "错误",
        description: "合约地址格式错误",
      });
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
        notification.error({
          message: "警告",
          description: "该代币未暴露获取精度方法 精度需你手动确认填写",
        });
        formData.precision = 18;
      }
    } else {
      notification.error({
        message: "警告",
        description: "未查询到该代币的ABI 精度需你手动确认填写",
      });
    }
  }
};

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

const onSubmit = async () => {
  if (!formData.receiveAddress) {
    message.warning("请填写转账地址");
    return;
  }
  if (
    formData.currency === 1 &&
    !tronWeb.value.isAddress(formData.contractAddress)
  ) {
    notification.error({
      message: "错误",
      description: "合约地址格式错误",
    });
    return;
  }
  try {
    isResult.value = false;
    fullscreenLoading.value = true;
    if (formData.way === 0) {
      await transaction(formData.receiveAddress);
    } else if (formData.way === 1) {
      const receiveAddress = dealAddresses(formData.receiveAddress);
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
    console.log("币已经全部发完了");
    notification.open({
      message: "发币结果",
      description: "币已经全部发完了",
      duration: 0,
    });
  } catch (error) {
    console.error(error);
  } finally {
    isResult.value = true;
    fullscreenLoading.value = false;
  }
};

const uzipAddress = (str) => {
  if (!str) return "-";
  return str.substr(0, 8) + "..." + str.substr(-4, 4);
};

const clearFormData = () => {
  formData.receiveAddress = undefined;
};

const reload = () => {
  window.location.reload();
};

watch(
  () => [formData.currency, formData.way],
  () => {
    clearFormData();
  }
);

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
.button-send {
  width: 40%;
}

.ant-form-item-label {
  font-weight: bold;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
}

.button-wrapper {
  display: flex;
  justify-content: space-around;
  align-items: center;
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

.ml-2 {
  margin-left: 10px;
}

.shake {
  animation: shake 1000ms ease-in-out infinite;
}

.rate {
  font-size: 14px;
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
