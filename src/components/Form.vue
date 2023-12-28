<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-12-14 23:39
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from 'vue';
  import { bitable, FieldType } from '@lark-base-open/js-sdk';
  import { ElMessage } from 'element-plus';
  import { addressCodeMap } from '@/addressCodeMap';

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

  // 根据身份证生成生日的信息
  function extractBirthdayAndTimestamp(idCard) {
    const year = parseInt(idCard.substring(6, 10), 10);
    const month = parseInt(idCard.substring(10, 12), 10);
    const day = parseInt(idCard.substring(12, 14), 10);
    const timestamp = new Date(year, month - 1, day).getTime();

    return {
      year,
      month,
      day,
      timestamp,
    };
  }

  function calculateAgeFromTimestamp(birthTimestamp) {
    const currentDate = new Date();
    const birthDate = new Date(birthTimestamp);

    const currentYear = currentDate.getFullYear();
    const birthYear = birthDate.getFullYear();

    let age = currentYear - birthYear;

    // 检查生日是否已经过了今年
    const currentMonth = currentDate.getMonth();
    const birthMonth = birthDate.getMonth();

    if (currentMonth < birthMonth || (currentMonth === birthMonth && currentDate.getDate() < birthDate.getDate())) {
      age--;
    }

    return age;
  }

  /**
   * @desc  : 校验身份证号码合法性
   * @param  { string } idCard 身份证号码
   * @return { object } 包含验证结果和可能的错误信息的对象
   */
  function checkIdCard(idCard) {
    // 结果对象，包含是否合法和错误信息
    const result = {
      isValid: false,
      errors: [],
    };

    // 校验身份证号码长度是否为18位（包括校验码）
    if (idCard.length > 18) {
      return {
        isValid: false,
        errorMessage: '长度大于18位',
      };
    }

    if (idCard.length < 18) {
      return {
        isValid: false,
        errorMessage: '长度小于18位',
      };
    }

    const year = parseInt(idCard.substr(6, 2));
    const isValidYear = year === 18 || year === 19 || year === 20;
    // 校验年份是否合理
    if (!isValidYear) {
      result.errors.push('年份不合理');
    }

    const birthMonth = parseInt(idCard.substr(10, 2));
    const isValidBirthMonth = birthMonth > 0 && birthMonth < 13;
    // 校验月份是否合理
    if (!isValidBirthMonth) {
      result.errors.push('月份不合理');
    }

    const birthDay = parseInt(idCard.substr(12, 2), 10);
    const isValidBirthDay = birthDay > 0 && birthDay < 32;
    // 校验日期是否合理
    if (!isValidBirthDay) {
      result.errors.push('日期不合理');
    }

    // 权重因子和校验码映射
    const weightedFactors = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2];
    const checkCodeMap = [1, 0, 'X', 9, 8, 7, 6, 5, 4, 3, 2];

    // 计算前17位的加权和
    let sum = 0;
    for (let i = 0; i < 17; i++) {
      sum += parseInt(idCard.charAt(i), 10) * weightedFactors[i];
    }

    // 计算校验码
    const mod = sum % 11;
    const checkCode = checkCodeMap[mod];

    // 校验身份证号码校验码是否正确
    const isValidCheckCode = idCard.charAt(17) === checkCode + '';
    if (!isValidCheckCode && isValidYear && isValidBirthMonth && isValidBirthDay) {
      result.errors.push('其他类型格式错误');
    }

    // 正则表达式校验身份证号码格式
    const regExp = /^[1-9]\d{5}(19|20)\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])\d{3}[\dX]$/;

    if (!regExp.test(idCard) && isValidCheckCode && isValidYear && isValidBirthMonth && isValidBirthDay) {
      result.errors.push('其他类型格式错误');
    }

    // 如果没有错误，将 isValid 设置为 true
    if (result.errors.length === 0) {
      result.isValid = true;
    }

    // 返回结果对象
    return {
      isValid: result.isValid,
      errorMessage: result.errors.join('、'),
    };
  }

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

    const hasCheck = tableMetaList.find((item) => item.name === '身份证号码格式错误');
    await asyncTasks.push(judgeCreate(hasCheck, '身份证号码格式错误', 'Text', generateCheckRow));

    const conditions = [
      { value: birthday.value, name: '生日', type: 'DateTime', func: generateBirthdayRow },
      { value: age.value, name: '年龄', type: 'Number', func: generateAgeRow },
      { value: sex.value, name: '性别', type: 'Text', func: generateSexRow },
      { value: constellation.value, name: '星座', type: 'Text', func: generateConstellationRow },
      { value: animal.value, name: '生肖', type: 'Text', func: generateAnimalRow },
      { value: address.value, name: '籍贯', type: 'Text', func: generateAddressRow },
    ];

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
   * @desc  : 生成身份证号码格式错误列
   */
  async function generateCheckRow() {
    const field = await table.getField('身份证号码格式错误');

    let _list = [];
    for (const record of recordList) {
      const id = record.id;

      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? '' : checkIdCard(val[0]?.text).errorMessage,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  /**
   * @desc  : 生成生日列
   */
  async function generateBirthdayRow() {
    const field = await table.getField('生日');
    const view = await table.getActiveView();
    await view.setFieldWidth(field.id, 100);

    await field.setDateFormat(dateFormat.value);

    let _list = [];
    for (const record of recordList) {
      const id = record.id;

      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? extractBirthdayAndTimestamp(val[0]?.text).timestamp : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  /**
   * @desc  : 生成年龄列
   */
  async function generateAgeRow() {
    const field = await table.getField('年龄');
    await field.setFormatter('0');

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

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid
            ? calculateAgeFromTimestamp(extractBirthdayAndTimestamp(val[0]?.text).timestamp)
            : null,
        },
      });
    }

    await table.setRecords(_list);
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

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? getGenderByIdCard(val[0]?.text) : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getGenderByIdCard(idCard) {
    // 获取身份证号码中的第17位数字
    const checkCode = parseInt(idCard.charAt(16), 10);

    // 判断奇偶性来确定性别
    return checkCode % 2 === 0 ? '女' : '男';
  }

  /**
   * @desc  : 生成星座列
   */
  async function generateConstellationRow() {
    const field = await table.getField('星座');
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

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? getConstellationByIdCard(val[0]?.text) : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getConstellationByIdCard(idCard) {
    const birthdayStr = idCard.substring(6, 14);
    const birthday = new Date(
      parseInt(birthdayStr.substring(0, 4)),
      parseInt(birthdayStr.substring(4, 6)) - 1,
      parseInt(birthdayStr.substring(6, 8)),
    );

    const month = birthday.getMonth() + 1;
    const day = birthday.getDate();

    const constellations = [
      { name: '白羊座', startMonth: 3, startDay: 21, endMonth: 4, endDay: 19 },
      { name: '金牛座', startMonth: 4, startDay: 20, endMonth: 5, endDay: 20 },
      { name: '双子座', startMonth: 5, startDay: 21, endMonth: 6, endDay: 21 },
      { name: '巨蟹座', startMonth: 6, startDay: 22, endMonth: 7, endDay: 22 },
      { name: '狮子座', startMonth: 7, startDay: 23, endMonth: 8, endDay: 22 },
      { name: '处女座', startMonth: 8, startDay: 23, endMonth: 9, endDay: 22 },
      { name: '天秤座', startMonth: 9, startDay: 23, endMonth: 10, endDay: 23 },
      { name: '天蝎座', startMonth: 10, startDay: 24, endMonth: 11, endDay: 22 },
      { name: '射手座', startMonth: 11, startDay: 23, endMonth: 12, endDay: 21 },
      { name: '摩羯座', startMonth: 12, startDay: 22, endMonth: 1, endDay: 19 },
      { name: '水瓶座', startMonth: 1, startDay: 20, endMonth: 2, endDay: 18 },
      { name: '双鱼座', startMonth: 2, startDay: 19, endMonth: 3, endDay: 20 },
    ];

    for (const constellation of constellations) {
      if (
        (month === constellation.startMonth && day >= constellation.startDay) ||
        (month === constellation.endMonth && day <= constellation.endDay)
      ) {
        return constellation.name;
      }
    }

    // 如果没有匹配的星座，返回空字符串或其他默认值
    return '';
  }

  /**
   * @desc  : 生成生肖列
   */
  async function generateAnimalRow() {
    const field = await table.getField('生肖');
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

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? getChineseZodiacByIdCard(val[0]?.text) : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getChineseZodiacByIdCard(idCard) {
    // 获取身份证中的年份
    const year = parseInt(idCard.substring(6, 10));

    // 生肖映射表
    const zodiacMap = ['猴', '鸡', '狗', '猪', '鼠', '牛', '虎', '兔', '龙', '蛇', '马', '羊'];

    // 计算生肖的索引
    const zodiacIndex = year % 12;

    // 返回对应的生肖
    return zodiacMap[zodiacIndex];
  }

  /**
   * @desc  : 生成籍贯列
   */
  async function generateAddressRow() {
    const field = await table.getField('籍贯');

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: checkIdCard(val[0]?.text).isValid ? getNativePlaceByIdCard(val[0]?.text) : null,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getNativePlaceByIdCard(idCard) {
    const provinceCode = idCard.substring(0, 2);
    const countyCode = idCard.substring(0, 6);

    // 根据省和县行政区划代码获取地址
    const province = addressCodeMap.province[provinceCode] || '未知省份';
    const county = addressCodeMap.county[countyCode] || '未知县区';

    return `${province}${addressFormat.value}${county}`;
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
      <div class="switch-tip">生日</div>
      <el-switch v-model="birthday" />
    </div>
    <div
      v-if="birthday"
      class="birthday"
    >
      <div class="title title-birthday">生日格式</div>
      <div>
        <el-select
          v-model="dateFormat"
          placeholder="请选择生日格式"
        >
          <el-option
            v-for="item in dateFormatList"
            :key="item.value"
            :label="item.name"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>
    <div class="switch">
      <div class="switch-tip">年龄</div>
      <el-switch v-model="age" />
    </div>

    <div class="switch">
      <div class="switch-tip">性别</div>
      <el-switch v-model="sex" />
    </div>
    <div class="switch">
      <div class="switch-tip">星座</div>
      <el-switch v-model="constellation" />
    </div>
    <div class="switch">
      <div class="switch-tip">生肖</div>
      <el-switch v-model="animal" />
    </div>
    <div class="switch">
      <div class="switch-tip">籍贯</div>
      <el-switch v-model="address" />
    </div>
    <div
      v-if="address"
      class="birthday"
    >
      <div class="title title-birthday">籍贯格式</div>
      <el-select
        v-model="addressFormat"
        placeholder="请选择籍贯格式"
      >
        <el-option
          v-for="item in addressFormatList"
          :key="item.value"
          :label="item.name"
          :value="item.value"
        />
      </el-select>
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
