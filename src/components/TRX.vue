<template>
  <el-card>
    <template #header>
      <div class="card-header">
        <span>你的地址：{{ ownerAddress || "请链接钱包" }}</span>
        <el-space>
          <el-button @click="reload">刷新页面</el-button>
          <el-button v-if="!ownerAddress" type="primary" @click="linkWallet"
            >链接钱包</el-button
          >
        </el-space>
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
        <el-input
          placeholder="请输入合法的代币地址"
          v-model.trim="formData.contractAddress"
          @blur="contractAddressBlur"
        ></el-input>
      </el-form-item>
      <el-form-item label="代币精度" v-if="formData.currency === 1">
        <el-input-number
          placeholder="请输入代币精度"
          :min="1"
          :max="18"
          :precision="0"
          v-model="formData.precision"
        />
        <div>
          精度请填写 1-18 可保证代币转账数量与填写的数量一致
          我们会自动读取代币合约的ABI获取精度 如果没有ABI
          需要你手动确认填写(精度可在钱包查看代币详情获取)
        </div>
      </el-form-item>
      <el-form-item label="转账方式">
        <div class="card-header">
          <el-radio-group v-model="formData.way">
            <el-radio :label="0">单个转账</el-radio>
            <el-radio :label="1">批量转账</el-radio>
            <el-radio :label="2">不同数量批量转账</el-radio>
          </el-radio-group>
          <el-link
            type="primary"
            href="/txt/template.txt"
            download="不同数量批量转账模板.txt"
            target="_blank"
            >不同数量批量转账模板</el-link
          >
        </div>
      </el-form-item>
      <el-form-item label="转账数量" v-if="formData.way !== 2">
        <el-input-number v-model="formData.num" :min="0"></el-input-number>
      </el-form-item>
      <el-form-item v-if="formData.way === 0" label="收款人地址(单个地址)">
        <el-input v-model.trim="formData.receiveAddress"></el-input>
      </el-form-item>
      <el-form-item
        v-else-if="formData.way === 1"
        label="收款人地址(多个地址换行)"
      >
        <el-input
          v-model="formData.receiveAddress"
          placeholder="仅仅只需要多个地址换行"
          autosize
          type="textarea"
        ></el-input>
      </el-form-item>
      <el-form-item v-else label="收款人地址(多个地址换行)">
        <el-input
          v-model="formData.receiveAddress"
          placeholder="地址|数量  多个换行 可以从模板中复制过来"
          autosize
          type="textarea"
        ></el-input>
      </el-form-item>
      <el-form-item label="备注">
        <el-input v-model="formData.remark" placeholder="请输入备注"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button
          v-loading.fullscreen.lock="fullscreenLoading"
          v-show="isResult"
          type="primary"
          @click="onSubmit"
          :disabled="!ownerAddress"
          >发放</el-button
        >
      </el-form-item>
      <el-form-item>
        <el-table :data="tableData" style="width: 100%">
          <el-table-column prop="receiveAddress" label="地址" width="180" />
          <el-table-column prop="num" label="数量" width="180" />
          <el-table-column prop="result" label="发送结果" />
        </el-table>
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

const linkWallet = () => {
  if (window.tronWeb) {
    tronWeb.value = window.tronWeb;
    ownerAddress.value = tronWeb.value.defaultAddress.base58;
  } else {
    ElMessage.warning("请下载波宝钱包浏览器插件");
  }
};

const contractAddressBlur = async () => {
  if (formData.contractAddress) {
    if (
      formData.currency === 1 &&
      !tronWeb.value.isAddress(formData.contractAddress)
    ) {
      ElNotification.error({
        title: "错误",
        message: "合约地址格式错误",
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
        ElNotification.error({
          title: "警告",
          message: "该代币未暴露获取精度方法 精度需你手动确认填写",
        });
        formData.precision = 18;
      }
    } else {
      ElNotification.error({
        title: "警告",
        message: "未查询到该代币的ABI 精度需你手动确认填写",
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
  const unsignedTransaction = await tronWeb.transactionBuilder.addUpdateData(
    tx,
    formData.remark,
    "utf-8"
  );
  const signedTx = await tronWeb.value.trx.sign(unsignedTransaction);
  const broastTx = await tronWeb.value.trx.sendRawTransaction(signedTx);
  console.log("TRX发放结果: ", receiveAddress);
  console.log(broastTx);
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
    ElMessage.warning("请填写转账地址");
    return;
  }
  if (
    formData.currency === 1 &&
    !tronWeb.value.isAddress(formData.contractAddress)
  ) {
    ElNotification.error({
      title: "错误",
      message: "合约地址格式错误",
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
    ElNotification({
      title: "发币结果",
      message: "币已经全部发完了",
      duration: 0,
    });
  } catch (error) {
    console.error(error);
  } finally {
    isResult.value = true;
    fullscreenLoading.value = false;
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

// 生成长度
function padWithZeros(number = 1, length = 1) {
  let str = "1";
  for (let index = 0; index < length; index++) {
    str += "0";
  }
  return number * str;
}

const reload = () => {
  window.location.reload();
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

.ml-2 {
  margin-left: 10px;
}
</style>
