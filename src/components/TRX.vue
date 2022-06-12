<template>
  <el-card>
    <template #header>
      <div class="card-header">
        <span>你的地址：{{ ownerAddress || "请链接钱包" }}</span>
        <el-button v-if="!ownerAddress" type="primary" @click="linkWallet"
          >链接钱包</el-button
        >
      </div>
    </template>
    <el-form ref="form" :model="formData" label-width="260px">
      <el-form-item label="转账币种">
        <el-radio-group v-model="formData.currency">
          <el-radio :label="0">TRX</el-radio>
          <el-radio :label="1">代币</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item label="代币合约" v-if="formData.currency === 1">
        <el-input v-model="formData.contractAddress"></el-input>
      </el-form-item>
      <el-form-item label="转账方式">
        <el-radio-group v-model="formData.way">
          <el-radio :label="0">单个转账</el-radio>
          <el-radio :label="1">批量转账</el-radio>
          <el-radio :label="2">不同数量批量转账</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item label="转账数量" v-if="formData.way !== 2">
        <el-input-number v-model="formData.num" :min="0"></el-input-number>
      </el-form-item>
      <el-form-item v-if="formData.way === 0" label="收款人地址(单个地址)">
        <el-input v-model="formData.receiveAddress"></el-input>
      </el-form-item>
      <el-form-item v-else label="收款人地址(多个地址换行)">
        <el-input v-model="formData.receiveAddress" type="textarea"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button
          v-show="isResult"
          type="primary"
          @click="onSubmit"
          :disabled="!ownerAddress"
          >发放</el-button
        >
      </el-form-item>
    </el-form>
  </el-card>
</template>

<script setup>
import { ElMessage, ElNotification } from "element-plus";
import { onMounted, reactive, ref, watch } from "vue";

const CONTANS = 1000000;
const ownerAddress = ref();
const isResult = ref(true);
const tronWeb = ref(null);

const formData = reactive({
  contractAddress: undefined,
  way: 1,
  num: 0.2,
  currency: 1,
  receiveAddress: undefined,
});

const linkWallet = () => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
    ownerAddress.value = tronWeb.value.defaultAddress.base58;
  } else {
    ElMessage.warning("请下载波宝钱包浏览器插件");
  }
};

const dealAddresses = (str) =>
  str
    .replace(/(\r\n)|(\n)/g, ",")
    .split(",")
    .filter((v) => v && v.trim());

// trx转账交易
const transactionTrx = async (receiveAddress) => {
  const tx = await tronWeb.value.transactionBuilder.sendTrx(
    receiveAddress,
    formData.num * CONTANS,
    ownerAddress.value
  );
  const signedTx = await tronWeb.value.trx.sign(tx);
  const broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  console.log("TRX发放结果: ", receiveAddress);
  console.log(broastTx);
};

/**
 * 代币转账
 * @param {String} receiveAddress 需要接收的地址
 * @param {Number} num  如果有 num ，代表自定义数量转账
 */
const transactionToken = async (receiveAddress, num = 0) => {
  const parameter = [
    { type: "address", value: receiveAddress },
    {
      type: "uint256",
      value: num ? parseInt(num * CONTANS) : formData.num * CONTANS,
    },
  ];

  const tx = await tronWeb.value.transactionBuilder.triggerSmartContract(
    formData.contractAddress,
    "transfer(address,uint256)",
    {},
    parameter,
    ownerAddress.value
  );

  const signedTx = await tronWeb.value.trx.sign(tx.transaction);
  const broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  const contract = await tronWeb.value.contract().at(formData.contractAddress);
  await contract.decimals().call();

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

// 判断转代币还是TRX
const transaction = async (receiveAddress) => {
  if (formData.currency === 0) {
    await transactionTrx(receiveAddress);
  } else {
    await transactionToken(receiveAddress);
  }
};

const onSubmit = async () => {
  if (!formData.receiveAddress) {
    ElMessage.warning("请填写转账地址");
    return;
  }
  try {
    isResult.value = false;
    if (formData.way === 0) {
      await transaction(formData.receiveAddress);
    } else if (formData.way === 1) {
      const receiveAddress = dealAddresses(formData.receiveAddress);
      for (let index = 0; index < receiveAddress.length; index++) {
        await transaction(receiveAddress[index]);
      }
    } else {
      const receiveAddress = dealAddresses(formData.receiveAddress);
      for (let index = 0; index < receiveAddress.length; index++) {
        const splitAddress = receiveAddress[index].split("\t");
        await transactionToken(splitAddress[0], splitAddress[1]);
      }
    }
    console.log("币已经全部发完了");
    ElNotification({
      title: "发币结果",
      message: "币已经全部发完了",
      duration: 0,
    });
  } catch (error) {
    console.error(error);
  } finally {
    isResult.value = true;
  }
};

watch(
  () => [formData.currency, formData.way],
  () => {
    clearFormData();
  }
);

const clearFormData = () => {
  formData.receiveAddress = undefined;
};

onMounted(() => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
  }
});
</script>

<style>
.card-header {
  display: flex;
  justify-content: space-between;
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
</style>
