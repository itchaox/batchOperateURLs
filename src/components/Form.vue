<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-12-30 10:06
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from 'vue';
  import { bitable, FieldType } from '@lark-base-open/js-sdk';
  import { ElMessage } from 'element-plus';
  import { addressCodeMap } from '@/addressCodeMap';

  import axios from 'axios';

  async function checkWebsite(url) {
    try {
      const response = await axios.get(url, {
        headers: {
          'Access-Control-Allow-Origin': '*', // 允许所有域访问，不建议在生产环境中使用
        },
        validateStatus: (status) => status >= 200 && status < 300,
      });

      if (response.status >= 200 && response.status < 300) {
        console.log(`${url} 是有效的网址`);
      } else {
        console.log(`${url} 返回状态码为 ${response.status}，可能是失效的网址`);
      }
    } catch (error) {
      console.error(`请求 ${url} 时发生错误:`, error.message);
    }
  }

  // 示例：检测一个网站是否有效
  checkWebsite('https://open.feishu.cn');

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  let dateFormatList = [
    {
      name: '格式: 01-30',
      value: 'MM-dd',
    },
    {
      name: '格式: 2021/01/30',
      value: 'yyyy/MM/dd',
    },
    {
      name: '格式: 2021-01-30',
      value: 'yyyy-MM-dd',
    },
    {
      name: '格式: 01/30/2021',
      value: 'MM/dd/yyyy',
    },
    {
      name: '格式: 30/01/2021',
      value: 'dd/MM/yyyy',
    },
  ];
  let dateFormat = ref('MM-dd');

  let table;
  let recordList;
  let recordIds;
  let tableMetaList;

  const age = ref(false);
  const sex = ref(false);
  const birthday = ref(false);
  const constellation = ref(false);
  const animal = ref(false);
  const address = ref(false);

  const isLoading = ref(false);

  let addressFormatList = [
    {
      name: '格式: 湖南长沙',
      value: '',
    },
    {
      name: '格式: 湖南-长沙',
      value: '-',
    },
    {
      name: '格式: 湖南_长沙',
      value: '_',
    },
    {
      name: '格式: 湖南/长沙',
      value: '/',
    },
  ];
  let addressFormat = ref('-');

  onMounted(async () => {
    table = await base.getActiveTable();
    recordList = await table.getRecordList();
    recordIds = await table.getRecordIdList(); // 获取所有记录 id

    tableMetaList = await table.getFieldMetaList();
    fieldOptions.value = tableMetaList.filter((item) => item.type === 1);
  });

  /**
   * @desc  : 确认按钮
   */
  async function confirm() {
    if (!fieldId.value) {
      ElMessage({
        type: 'error',
        message: '请选择身份证号码列!',
        duration: 1500,
      });
      return;
    }

    ElMessage({
      type: 'success',
      message: '开始生成数据~',
      duration: 1500,
    });

    isLoading.value = true;

    const asyncTasks = [];

    const conditions = [{ value: sex.value, name: '性别', type: 'Text', func: generateSexRow }];

    conditions.forEach(({ value, name, type, func }) => {
      if (value) {
        const hasCondition = tableMetaList.find((item) => item.name === name);
        asyncTasks.push(judgeCreate(hasCondition, name, type, func));
      }
    });

    // 使用 Promise.all 确保所有异步操作完成
    await Promise.all(asyncTasks);

    ElMessage({
      type: 'success',
      message: '数据生成结束!',
      duration: 1500,
    });

    isLoading.value = false;
  }

  /**
   * @desc  : 判断是否生成
   * @param  {any} has 是否有字段
   * @param  {any} label 字段名
   * @param  {any} type 字段类型
   * @param  {any} fn 生成函数
   */
  async function judgeCreate(has, label, type, fn) {
    if (!has) {
      await table.addField({ type: FieldType[type], name: label });
    }

    await fn();
  }

  /**
   * @desc  : 生成性别列
   */
  async function generateSexRow() {
    const field = await table.getField('性别');
    const view = await table.getActiveView();
    await view.setFieldWidth(field.id, 20);

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // console.log(await check404WithImage(val[0].link || val[0].text), '22222');
      // console.log(await checkWebsite(val[0].link || val[0].text), '22222');

      console.log(await checkLink(val[0].link || val[0].text), '333');

      // let _code = await check404WithImage(val[0].link || val[0].text);
      // let _code = await checkWebsite(val[0].link || val[0].text);

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          // [field.id]: checkIdCard(val[0]?.text).isValid ? getGenderByIdCard(val[0]?.text) : null,
          // [field.id]: true ? getGenderByIdCard(val[0]?.text) : null,
          [field.id]: _code === '404' ? '链接可能已经失效' : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  async function checkLink(url) {
    linkCheck(url, function (err, result) {
      if (err) {
        console.error(err);
        return;
      }
      console.log(`${result.link} is ${result.status}`);
    });
    console.log(results);
  }

  function getGenderByIdCard(idCard) {
    // 获取身份证号码中的第17位数字
    const checkCode = parseInt(idCard.charAt(16), 10);

    // 判断奇偶性来确定性别
    return checkCode % 2 === 0 ? '女' : '男';
  }
</script>

<template>
  <div>
    <div class="tip">
      <div class="tip-text tip-title">操作步骤:</div>
      <div class="tip-text">1. 必须选择身份证号码列</div>
      <div class="tip-text">2. 按需选择需要获取的信息列</div>
      <div class="tip-text">3. 点击【确认生成】按钮即可</div>
      <div class="tip-text tip-info">Tips：</div>
      <div class="tip-text">自动校验身份证号码格式，并生成错误列</div>
    </div>
    <div class="idCard">
      <div class="title">身份证号码列</div>
      <div>
        <el-select
          v-model="fieldId"
          placeholder="请选择身份证号码列"
        >
          <el-option
            v-for="item in fieldOptions"
            :key="item.id"
            :label="item.name"
            :value="item.id"
          />
        </el-select>
      </div>
    </div>

    <div class="row-title">
      <span> 请选择需要生成的 </span>
      <span class="high">信息列: </span>
    </div>
    <div class="row-tip">Tips: 表格中无对应信息列, 则会自动生成</div>

    <div class="switch">
      <div class="switch-tip">性别</div>
      <el-switch v-model="sex" />
    </div>

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
      :loading="isLoading"
      >确认生成</el-button
    >
  </div>
</template>

<style scoped>
  .tip {
    color: #8f959e;
    font-size: 12px;
    margin-bottom: 24px;
    .tip-text {
      margin-bottom: 6px;
    }

    .tip-info {
      margin-top: 12px;
    }

    .tip-title {
      font-size: 14px;
      margin-bottom: 8px;
    }
  }

  .row-title {
    margin-bottom: 6px;
  }

  .row-tip {
    color: #8f959e;
    font-size: 12px;
    margin-bottom: 14px;
  }

  .idCard {
    margin-bottom: 18px;
  }

  .title {
    font-size: 16px;
    font-weight: 700;
    color: rgb(31, 35, 41);
    margin-bottom: 6px;
  }

  .btn {
    margin-top: 10px;
  }

  .birthday {
    margin-bottom: 14px;
    .title-birthday {
      font-size: 14px;
    }
  }

  .switch {
    display: flex;
    align-items: center;
    margin-bottom: 12px;

    .switch-tip {
      margin-right: 5px;
    }
  }

  .high {
    color: dodgerblue;
  }
</style>
