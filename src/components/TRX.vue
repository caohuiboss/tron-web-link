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
      <el-form-item label="代币合约">
        <el-input v-model="formData.contractAddress"></el-input>
      </el-form-item>
      <el-form-item label="转账方式">
        <el-radio-group v-model="formData.way">
          <el-radio :label="0">单个转账</el-radio>
          <el-radio :label="1">批量转账</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item label="转账数量">
        <el-input-number v-model="formData.num" :min="0"></el-input-number>
      </el-form-item>
      <el-form-item v-if="formData.way === 0" label="收款人地址(单个地址)">
        <el-input v-model="formData.receiveAddress"></el-input>
      </el-form-item>
      <el-form-item v-else label="收款人地址(多个地址换行)">
        <el-input v-model="formData.receiveAddress" type="textarea"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="onSubmit" :disabled="!ownerAddress"
          >发放</el-button
        >
      </el-form-item>
    </el-form>
  </el-card>
</template>

<script setup>
import { ElMessage } from "element-plus";
import { onMounted, reactive, ref } from "vue";

const CONTANS = 1000000;
const ownerAddress = ref();
const tronWeb = ref(null);

const formData = reactive({
  contractAddress: "TRTSKHtGuvX2rQeQLgtkfya1MYgidHmsmd",
  way: 1,
  num: 0.2,
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

const onSubmit = async () => {
  if (!formData.receiveAddress) {
    ElMessage.warning("请填写转账地址");
    return;
  }
  if (formData.way === 0) {
    transactionToken(formData.receiveAddress);
  } else {
    const receiveAddress = dealAddresses(formData.receiveAddress);
    for (let index = 0; index < receiveAddress.length; index++) {
      await transactionToken(receiveAddress[index]);
    }
  }
};

const sleep = async (time = 5000) => {
  return new Promise((resolve) => setTimeout(resolve, time));
};

const transactionToken = async (receiveAddress) => {
  const parameter = [
    { type: "address", value: receiveAddress },
    { type: "uint256", value: formData.num * CONTANS },
  ];
  var tx = await tronWeb.value.transactionBuilder.triggerSmartContract(
    formData.contractAddress,
    "transfer(address,uint256)",
    {},
    parameter,
    ownerAddress.value
  );
  var signedTx = await tronWeb.value.trx.sign(tx.transaction);
  var broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  console.log("发放结果: ", receiveAddress);
  console.log(broastTx)
  let contract = await tronWeb.value.contract().at(formData.contractAddress);
  let result1 = await contract.decimals().call();
  console.log(result1);
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
