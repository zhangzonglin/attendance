<script setup lang="ts">
import { ElButton, ElMessage, ElInput, ElTable, ElTableColumn, ElPagination, ElTag, ElTooltip, ElIcon, ElSelect, ElOption } from 'element-plus'
import { ref, reactive } from 'vue'
// @ts-ignore
import FileSaver from 'file-saver'
import Excel from 'exceljs'
import moment from 'moment'
import clipboard3 from 'vue-clipboard3'


let base_url = '/assets/excel/prod/attendance.xlsx'
if (process.env.NODE_ENV === 'production') {
  base_url = '/ilan/assets/excel/attendance.xlsx'
}
console.log('base_url:', base_url)

const template_excel_url = base_url
const excelInput = ref<HTMLInputElement>();
const loading = ref(false)  //文件上传loading
const parsed = ref(false)  //是否已经解析文件并生成数据
const attendance_date = ref(null)  //打卡月份
const record = ref([])  //解析出来的打卡数据，页面展示用
const missing = ref([])  //未找到签到数据人员列表，页面展示用
const cell_prefix_index = 3  //表头行数
const column_number = 31  //天的总列数
const specified_work_hours = 9 * 60 //指定的上班时间，单位：分钟
let days_in_month = 0 //月份天数

const query = reactive({
  name: "",
  pageIndex: 1,
  pageSize: 15,
  absence: ''
});//姓名查询
const tableData = ref([]);
const pageTotal = ref(0);

const upload = (rawFile: File) => {
  if (!/\.(xls|xlsx)$/.test(rawFile.name.toLowerCase())) {
    console.log('上传格式不正确，请上传xls或者xlsx格式')
    ElMessage.error('上传格式不正确，请上传xls或者xlsx格式')
    return false
  }
  if (excelInput.value) {
    excelInput.value.value = ''
  }
  //重置数据
  record.value = []
  missing.value = []
  attendance_date.value = null
  days_in_month = 0
  query.name = ''
  tableData.value = []
  pageTotal.value = 0
  parsed.value = false
  //上传文件状态标志,防止重复上传
  loading.value = true
  readExcel(rawFile).then((sign_data: any) => {
    console.log('sign_data:', sign_data) // 转换成json的数据

    var xhr = new XMLHttpRequest();
    xhr.open('get', template_excel_url, true);
    xhr.responseType = 'arraybuffer';

    xhr.onload = function (e) {
      if (xhr.status == 200) {
        var data = new Uint8Array(xhr.response)
        var workbook = new Excel.Workbook();
        workbook.xlsx.load(data).then(function () {
          var worksheet = workbook.getWorksheet(1);
          let first_row = worksheet.getRow(1)
          let second_row = worksheet.getRow(2)
          if (days_in_month == 0 || attendance_date.value == null) {
            throw new Error("读取Excel失败，月份天数未初始化")
          }
          //初始化表头
          let first_day_of_month = (attendance_date.value as any).startOf('month') //获取该月月初的日期
          for (let index = 1; index <= column_number; index++) {
            if (index <= days_in_month) {
              //index 除于7的余数
              second_row.getCell(cell_prefix_index + index).value = parseWeekByNumber((index - 1 + first_day_of_month.day()) % 7)
            } else {
              first_row.getCell(cell_prefix_index + index).value = ''
              second_row.getCell(cell_prefix_index + index).value = ''
              worksheet.getColumn(cell_prefix_index + index).hidden = true
            }
          }

          //填充数据
          worksheet.eachRow((row, rowNumber) => {
            if (rowNumber < 4) {
              return
            }
            let name = row.getCell(3).value as string
            if (name == null || name == '') {
              return
            }
            //name 去除所有不可见字符
            name = name.replace(/[\u0000-\u0019]/g, '')
            if (!sign_data.hasOwnProperty(name)) {
              (missing.value as any).push(name)
              return
            }
            populateRow(name, sign_data, row)
          })
          if (missing.value.length > 0) {
            let miss_names = missing.value.join(',')
            console.log('打卡原始记录中没有找到员工： ', miss_names)
            ElMessage({
              message: '原始打卡记录中没有找到如下员工： ' + miss_names,
              type: 'warning',
              duration: 8000
            })

          }
          //列出sign_data对象属性，如果还有的话，说明有模板上未列出的员工，添加进去... 
          //todo 添加代码弄成函数
          for (let key in sign_data) {
            console.log('key:', key)
            const last_row = worksheet.lastRow;
            // console.log('last_row:', last_row)
            if (!last_row) {
              console.log('last_row is null')
              ElMessage.error('未找到最后一行')
              return
            }
            worksheet.insertRow(last_row.number, [], 'i+')
            let new_row = worksheet.getRow(last_row.number)
            new_row.getCell(3).value = key
            populateRow(key, sign_data, new_row)
          }

          pageTotal.value = record.value.length
          //初始化分页数据
          handleSearch()
          console.log('record.value:', record.value)
          workbook.xlsx.writeBuffer().then(function (buffer) {
            var blob = new Blob([buffer], { type: 'application/vnd.ms-excel;charset=utf-8' });
            let file_name = '考勤.xlsx'
            if (attendance_date.value != null) {
              file_name = `${(attendance_date.value as any).format('YYYY年MM月')}考勤.xlsx`
            }

            FileSaver.saveAs(blob, file_name);
          });
          parsed.value = true

        });
      } else {
        throw new Error('读取模板失败')
      }
    };
    xhr.send();
  }).catch(err => {
    parsed.value = false
    console.log(err)
    ElMessage.error('上传失败，请联系月神丘比特:' + err)
  }).finally(() => {
    loading.value = false
  })
}

const populateRow = (name: any, sign_data: any, row: any) => {
  var employee = (sign_data as any)[name]
  let lack_sign_days = 0
  for (let index = 1; index <= days_in_month; index++) {
    let sign_day = employee.sign_arrary[index - 1]
    let text = '休'
    if (sign_day !== undefined && sign_day !== null) {
      if (sign_day.flag == 1) {
        text = '缺卡'
        lack_sign_days++
      } else {
        text = '√'
      }

      //创建页面展示数据,并保存在数组里
      let row_data = {
        name: name,
        day: index,   //日期
        sign_in: '', //上班打卡时间              
        sign_out: '',   //下班打卡时间
        flag: sign_day.flag, //打卡标识
        sign_text: text, //打卡文本
        sign_days: employee.sign_days_num, //该月出勤总天数
        meal_supplement: sign_day.meal_supplement, //今日是否餐补
        supplement_days: employee.supplement_days_num, //该月餐补总天数
        makeup_num: sign_day.makeup_num, //今日补卡次数
        total_makeup_num: employee.total_makeup_num, //该月补卡总次数
        remark: sign_day.remark, //备注
        work_hours: sign_day.work_hours, //工作时长(分钟)
        work_hours_text: '', //工作时长(文本)
        absence_hours: 0, //缺勤时长(分钟)
        absence_hours_text: '', //缺勤时长(文本)
      }

      row_data.work_hours_text = parseMin2String(sign_day.work_hours)

      //查看工作时长是否超过14小时，超过14则认为数据解析有问题，给老婆预览
      if (sign_day.work_hours > 14 * 60) {
        row_data.remark += ' 工作' + row_data.work_hours_text + ','
      }
      if (row_data.remark != '') {
        row_data.remark += ' 跪请女王大人核实!'
      }

      //计算缺勤时长
      if (sign_day.work_hours > 0 && sign_day.work_hours < specified_work_hours) {
        row_data.absence_hours = specified_work_hours - sign_day.work_hours
        row_data.absence_hours_text = parseMin2String(row_data.absence_hours)
      }
      //打卡时间格式
      row_data.sign_in = sign_day.min_time.format('YYYY-MM-DD HH:mm:ss')
      if (sign_day.max_time != null) {
        row_data.sign_out = sign_day.max_time.format('YYYY-MM-DD HH:mm:ss')
      }

      (record.value as any).push(row_data)
    }
    row.getCell(cell_prefix_index + index).value = text
  }
  //考勤扣款
  if (employee.total_makeup_num > 3) {
    lack_sign_days += employee.total_makeup_num - 3
  }
  if (lack_sign_days > 0) {
    row.getCell(cell_prefix_index + days_in_month + 5).value = (lack_sign_days * 20).toFixed(2);
  }

  //实际出勤
  row.getCell(cell_prefix_index + days_in_month + 9).value = employee.sign_days_num
  //餐补天数
  if (employee.supplement_days_num > 0) {
    row.getCell(cell_prefix_index + days_in_month + 12).value = employee.supplement_days_num
  }
  row.commit()
  delete sign_data[name]

}

//打卡记录最大也就一天24小时，所以不用考虑跨天的情况
const parseMin2String = (min: number) => {
  let work_time = moment.duration(min, 'minutes')
  let minute = work_time.minutes()
  let hour = work_time.hours()
  let text = ''
  if (hour > 0) {
    text += hour + 'h'
  }
  if (minute > 0) {
    text += minute + 'm'
  }
  return text
}

const parseWeekByNumber = (number: number) => {
  switch (number) {
    case 1:
      return '一'
    case 2:
      return '二'
    case 3:
      return '三'
    case 4:
      return '四'
    case 5:
      return '五'
    case 6:
      return '六'
    case 0:
      return '日'
    default:
      return number
  }
}



const readExcel = (rawFile: File) => {
  return new Promise((resolve, reject) => {
    const wb = new Excel.Workbook();
    const reader = new FileReader()
    reader.readAsArrayBuffer(rawFile)
    reader.onload = () => {
      try {
        const buffer = reader.result as Buffer;
        wb.xlsx.load(buffer).then(() => {
          const sheet = wb.getWorksheet(1)
          parseSheet(sheet).then(res => {
            resolve(res)
          }).catch(err => {
            reject(err)
          })
        })
      } catch (err) {
        reject(err)
      }
    }
  })
}


//函数，解析行数据
const parseSheet = (sheet: any) => {
  return new Promise((resolve, reject) => {
    try {
      let parsed_data = {} //统计的打卡数据
      let name_cell_index = 0 //姓名单元格索引
      let date_cell_index = 0 //日期单元格索引
      let source_cell_index = 0 //数据来源单元格索引
      sheet.eachRow((row: any, rowIndex: any) => {
        if (rowIndex === 1) {
          row.eachCell((cell: any, colIndex: any) => {
            if (cell.value.trim() === '姓名') {
              name_cell_index = colIndex
            } else if (cell.value.trim() === '打卡时间') {
              date_cell_index = colIndex
            } else if (cell.value.trim() === '数据来源') {
              source_cell_index = colIndex
            }
          })
          return //skip  header row
        }
        if (name_cell_index === 0 || date_cell_index === 0 || source_cell_index === 0) {
          reject('获取表头【姓名】【打卡时间】【数据来源】列的位置索引失败 ')
        }
        let cell_date_str = row.getCell(date_cell_index).value//日期
        if (cell_date_str != '' && cell_date_str != null) {
          cell_date_str = cell_date_str.replace(/-/g, '/').replace(/\s+/g, ' ').trim()  //替换所有的-为/, 中间多个空格替换成1个
        }
        // console.log('cell_date_str', cell_date_str)
        let cell_date = moment(cell_date_str, 'YYYY/MM/DD HH:mm:ss', true) //日期
        if (!cell_date.isValid())
          reject('请检查第' + rowIndex + '行打卡时间的日期格式，日期：' + cell_date_str + ' 非YYYY/MM/DD HH:mm:ss格式 ')
        // console.log(row.values, rowIndex)
        let attendance_month = cell_date.get('month') + 1  //月份是零索引的，因此一月是月份 0
        // console.log('attendance_month', attendance_month)
        let attendance_day = cell_date.get('date')
        // console.log('attendance_day', attendance_day)

        let attendance_hour = cell_date.get('hour')
        if (days_in_month === 0) {
          days_in_month = cell_date.daysInMonth()
        }
        (attendance_date.value as any) ??= cell_date
        // console.log('attendance_date', attendance_date.value)
        let first_row_month = (attendance_date.value as any).get('month') + 1 //月份是零索引的，因此一月是月份 0

        if (first_row_month !== attendance_month) {
          attendance_date.value = null
          days_in_month = 0
          reject('请检查日期,数据月份不一致：' + first_row_month + '月与 ' + attendance_month + '月')
        }
        // console.log(attendance_day, attendance_month, days_in_month)

        //根据名字，获取考勤对象，如果不存在，则创建一个新的考勤对象
        let name = row.getCell(name_cell_index).value
        if (name === '' || name == null) {
          reject('第' + rowIndex + '行姓名为空')
        }
        name = name.replace(/[\u0000-\u0019]/g, '') //去除掉无效的字符
        if (parsed_data.hasOwnProperty(name)) {
          var employee_records = (parsed_data as any)[name]
        } else {
          var employee_records: any = {
            sign_arrary: new Array(days_in_month), //该月的每日打卡数组
            sign_days_num: 0,  //该月打卡天数（包括缺卡和正常打卡的）
            supplement_days_num: 0,  //该月餐补总天数
            total_makeup_num: 0,  //该月补卡总次数
          }
        }

        //确定该打卡记录对应的日期以及相应数组位置
        let arrary_index = attendance_day - 1
        // console.log('hour:', attendance_hour)
        if (attendance_hour < 7) { //如果是在早上7点之前，则认为是前一天的打卡(下班)记录,即数组索引arrary_index - 1
          if (attendance_day === 1) { //1号早上7点之前，则认为是上一个月的最后一天的打卡(下班)记录
            console.log('1号，并且是7点之前的打卡，跳过这条数据: ', row.values)
            return
          }
          arrary_index--
        }

        //根据日期索引，获取考勤对象的数组位置，如果不存在，则创建一个新的签到对象
        let sign_day = employee_records.sign_arrary[arrary_index]
        if (sign_day === undefined || sign_day === null) { //默认第一条数据的时间为最小时间，后面再跟其他数据比较
          sign_day = {
            min_time: cell_date,
            max_time: null,
            work_hours: 0, //工作时长(分钟)
            meal_supplement: false,
            makeup_num: 0, //今日补卡次数
            flag: 1, //1:打卡一次，2:打卡两次(及以上)
            remark: '' //备注（后续根据该字段是否为空，判断要不要把该数据展示在页面上，供老婆参考，比如时间凌晨6点的打卡记录）
          }
          employee_records.sign_days_num++
          employee_records.sign_arrary[arrary_index] = sign_day
        } else {
          //比较以及确定最小最大日期
          if (sign_day.max_time === null) { //说明以前只有一次打卡记录
            if (cell_date.isBefore(sign_day.min_time)) { //如果是早于最小时间，则更新最小时间和最晚时间
              let temp_date = sign_day.min_time
              sign_day.min_time = cell_date
              sign_day.max_time = temp_date
            } else {
              sign_day.max_time = cell_date
            }
          } else { //说明已有两次打卡记录
            if (cell_date.isBefore(sign_day.min_time)) { //如果是早于最小时间，则更新最小时间
              sign_day.min_time = cell_date
            } else if (cell_date.isAfter(sign_day.max_time)) { //如果是晚于最晚时间，则更新最晚时间
              sign_day.max_time = cell_date
            }
          }
          if (sign_day.min_time === null || sign_day.max_time === null) {
            reject('讲道理，此处逻辑执行不到，且当做测试吧,解析第' + rowIndex + '行时，最大最小时间至少一个为空')
          }
          //设置打卡状态标志位
          sign_day.flag = 2
          //设置工作时长(分钟)
          sign_day.work_hours = sign_day.max_time.diff(sign_day.min_time, 'minutes')
        }
        // sign_day.remark = '' //清空，避免某天有多条打卡数据的情况下有脏记录，新的数据会在下面重新计算一遍
        //补卡次数，每天的和该月总计的
        let source = row.getCell(source_cell_index).value //数据来源
        if (source !== null && source !== '' && source.trim() === '补打卡考勤') {
          sign_day.makeup_num++
          employee_records.total_makeup_num++
        }

        // console.log('sign_day:', sign_day)
        //计算今日是否餐补，和统计到这月餐补总天数，以及凌晨3点到7点打卡的添加到备注里
        //只有一次打卡记录，sign_day.min_time初始化即为cell_date,所以attendance_hour即min_time的时间
        if (sign_day.max_time == null) {
          //8点01分之前，则包括第二天凌晨的（7点之前，肯定超过晚上20点下班了）和今早8点之前(7点多，包括8点整)，19点之后则是晚上20点之后的（包括8点整）
          if (attendance_hour < 8 || (attendance_hour == 8 && cell_date.get('minutes') < 1) || attendance_hour > 19) {
            sign_day.meal_supplement = true
            employee_records.supplement_days_num++
          }
          //计算是否有凌晨特殊时间打卡的（凌晨4点之后7点之前的）
          if (attendance_hour < 7 && attendance_hour > 3) { //第一条数据的时间小于7点，则认为是下班的打卡记录
            sign_day.remark = '凌晨' + attendance_hour + '点打卡,行迹十分诡异,'
          }
        } else {
          let max_hour = sign_day.max_time.get('hour')
          let min_hour = sign_day.min_time.get('hour')
          //只有在本条打卡记录以前统计的今日无餐补的情况下，才会重新计算是否餐补（只要以前统计的确定餐补了，就算有新的最大或最小时间进来，肯定也是符合餐补规则的<8点01分or>=20点）
          if (!sign_day.meal_supplement && (max_hour < 8 || max_hour > 19 || min_hour < 8 || (min_hour == 8 && sign_day.min_time.get('minutes') < 1))) {
            sign_day.meal_supplement = true
            employee_records.supplement_days_num++
          }
          if (max_hour < 7 && max_hour > 3) { //打卡记录在凌晨4之后，7点之前的，特呈于老婆预览
            sign_day.remark = '凌晨' + max_hour + '点打卡,行迹诡异,'
          }
        }

        (parsed_data as any)[name] = employee_records
        // console.log('parsed_data:', parsed_data)
      })
      resolve(parsed_data)
    } catch (error) {
      console.log('error:', error)
      reject(error)
    }
  })
}

// 查询操作
const handleSearch = () => {
  query.pageIndex = 1
  query.name = query.name.trim()
  getData()
}
// 分页导航
const handlePageChange = (val: any) => {
  query.pageIndex = val
  getData()
}

let query_data = [] as any
let query_str = ''
// 获取表格数据
const getData = () => {
  try {
    console.log('query:', query)
    if (query.name == '' && query.absence == '') {
      pageTotal.value = record.value.length
      tableData.value = record.value.slice((query.pageIndex - 1) * query.pageSize, query.pageIndex * query.pageSize)
    } else {
      if (query.name + query.absence != query_str) { //查询条件变化了，重新查询
        query_str = query.name + query.absence
        if (query.name != '') {
          query_data = record.value.filter(item => {
            return (item as any).name.indexOf(query.name) > -1
          })
        }else{
          query_data = record.value
        }
        if (query.absence != '') {
          let line = 1
          if (query.absence == '2') {
            line = 60
          }
          //@ts-ignore
          query_data = query_data.filter((item) => {
            return item.absence_hours >= line
          })
          console.log('query_data:', query_data)
        }
      }
      pageTotal.value = query_data.length
      tableData.value = query_data.slice((query.pageIndex - 1) * query.pageSize, query.pageIndex * query.pageSize)
    }
  } catch (error) {
    console.log('error:', error)
    ElMessage.error("页面查询数据出错，联系老公 " + error)
  }
}


const handleClick = () => {
  const files = excelInput.value?.files
  if (!files) {
    ElMessage.error('请选择文件')
    return false
  }
  if (files.length !== 1) {
    ElMessage.error('Only support uploading one file!')
    return
  }
  upload(files[0])
}

const handleDrop = (e: DragEvent) => {
  e.stopPropagation()
  e.preventDefault()
  if (loading.value) {
    ElMessage.error('请等待上一次解析完成，或者直接联系哥哥~')
    return
  }
  if (!e.dataTransfer) return
  const files = e.dataTransfer.files
  if (files.length !== 1) {
    ElMessage.error('Only support uploading one file!')
    return
  }
  upload(files[0])
  e.stopPropagation()
  e.preventDefault()
}

const handleDragover = (e: DragEvent) => {
  e.stopPropagation()
  e.preventDefault()
  if (e.dataTransfer) {
    e.dataTransfer.dropEffect = 'copy'
  }
}

const handleUpload = () => {
  excelInput.value?.click()
}

//点击复制
const { toClipboard } = clipboard3();

const copy = async (val: any) => {
  try {
    await toClipboard(val);
    ElMessage.success("复制: " + val + ' 成功!!')
  } catch (error) {
    ElMessage.error("复制失败!!")
  }
};
</script>

<template>
  <div style="display:flex; justify-content: center;">
    <input ref="excelInput" class="excel-upload-input" type="file" accept=".xlsx, .xls" @change="handleClick">
    <div class="drop" @drop="handleDrop" @dragover="handleDragover" @dragenter="handleDragover">
      拖曳 excel 文件至此 or
      <el-button :loading="loading" style="margin-left:16px;" size="default" type="primary" @click="handleUpload">
        浏览上传
      </el-button>
    </div>
    <div class="note" v-show="missing.length > 0">原始打卡记录中没有找到如下员工：<span style="color:red">{{ missing.join(',') }}</span>
    </div>

  </div>
  <div style="clear:both"></div>
  <div v-show="parsed" class="container">
    <div class="handle-box">
      <el-select v-model="query.absence" placeholder="全部" class="handle-select mr10">
        <el-option key="1" label="全部" value=''></el-option>
        <el-option key="2" label="缺勤(是)" value='1'></el-option>
        <el-option key="3" label="缺勤(>1h)" value='2'></el-option>
      </el-select>
      <el-input v-model="query.name" placeholder="员工名" class="handle-input mr10" @keydown.enter="handleSearch">
      </el-input>
      <el-button type="primary" @click="handleSearch">
        <el-icon style="vertical-align: middle">
          <Search />
        </el-icon>
        <span style="vertical-align: middle">搜索</span>
      </el-button>
    </div>
    <el-table :data="tableData" :border="true" class="table" ref="multipleTable" header-cell-class-name="table-header">
      <el-table-column prop="name" align="center" width="50rem" slot="test" label="No.">
        <template #default="scope">
          {{ scope.$index + 1 }}
        </template>
      </el-table-column>
      <el-table-column align="center" width="100rem" label="员工名">
        <template #default="scope">
          <div style="cursor:pointer" @click="copy(scope.row.name)">
            {{ scope.row.name }}
          </div>
        </template>
      </el-table-column>
      <el-table-column align="center" width="70rem" label="日期">
        <template #default="scope">{{ scope.row.day }}号</template>
      </el-table-column>
      <el-table-column align="center" label="今日补卡(次)">
        <template #default="scope">
          <span v-if="scope.row.makeup_num > 0">{{ scope.row.makeup_num }}</span>
        </template>
      </el-table-column>
      <el-table-column align="center" label="今月补卡(次)">
        <template #default="scope">
          <span v-if="scope.row.total_makeup_num > 0">{{ scope.row.total_makeup_num }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="sign_days" align="center" width="130rem">
        <template #header>
          <el-tooltip placement="top" effect="dark" content="该日有打卡记录的，统统统计进来" raw-content>
            <span style="vertical-align: middle"> 实际出勤(天) <el-icon style="vertical-align: middle; color: #409EFC">
                <InfoFilled />
              </el-icon></span>
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column align="center" width="150rem">
        <template #header>
          <el-tooltip class="item" effect="dark" content="<div>1.凌晨7点以前的打卡，俱算作前一天的打卡</div>
            <div>2.即使缺勤，也标记为‘√’</div>" raw-content>
            <span style="vertical-align: middle"> 打卡状态 <el-icon style="vertical-align: middle; color: #409EFC">
                <InfoFilled />
              </el-icon></span>
          </el-tooltip>
        </template>
        <template #default="scope">
          <el-tag type="success">{{ scope.row.sign_text }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="sign_in" align="center" max-width="150rem" label="上班打卡">
      </el-table-column>
      <el-table-column prop="sign_out" align="center" max-width="150rem" label="下班打卡">
      </el-table-column>
      <el-table-column align="center" label="餐补天数">
        <template #default="scope">
          <span v-if="scope.row.supplement_days > 0">{{ scope.row.supplement_days }}</span>
        </template>
      </el-table-column>
      <el-table-column align="center" width="100rem">
        <template #header>
          <el-tooltip placement="top" effect="dark" content="早上7点到8点打卡(大于等于7点或小于8点1分)，或者晚上20点以后打卡(大于等于20点)" raw-content>
            <span style="vertical-align: middle"> 餐补 <el-icon style="vertical-align: middle; color: #409EFC">
                <InfoFilled />
              </el-icon></span>
          </el-tooltip>
        </template>
        <template #default="scope">
          <el-tag v-if="scope.row.meal_supplement" type="success">√</el-tag>
        </template>
      </el-table-column>
      <el-table-column align="center" width="130rem" label="工作时长">
        <template #default="scope">
          <span v-if="scope.row.work_hours > 0 && scope.row.work_hours < 11 * 60">{{ scope.row.work_hours_text
          }}</span>
          <el-tag v-else-if="scope.row.work_hours >= 11 * 60" type="warning">{{ scope.row.work_hours_text }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column align="center" width="150rem" label="缺勤时长(<9h)">
        <template #default="scope">
          <span v-if="scope.row.absence_hours > 0 && scope.row.absence_hours < 60">{{ scope.row.absence_hours_text
          }}</span>
          <el-tag v-else-if="scope.row.absence_hours >= 60" type="warning">{{ scope.row.absence_hours_text }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column align="center" width="250rem" :showOverflowTooltip="false">
        <template #header>
          <el-tooltip placement="top" effect="dark"
            content="凌晨6点打卡的都算作前天的打卡，可能导致今日的上班打卡被当做昨天的下班打卡，以致工作时长特别夸张，但凡本列不为空的日期，都要女王大人自己人工复查一遍原始打卡记录(尤其是<Strong>今天和前天</Strong>的打卡记录)"
            raw-content>
            <span style="vertical-align: middle"> 凌晨4~7点打卡or工作大于14h <el-icon
                style="vertical-align: middle; color: #409EFC">
                <InfoFilled />
              </el-icon></span>
          </el-tooltip>
        </template>
        <template #default="scope">
          <el-tag v-if="scope.row.remark != ''" type="danger">注意</el-tag>{{ scope.row.remark }}
        </template>
      </el-table-column>
    </el-table>
    <div class="pagination">
      <el-pagination background :current-page="query.pageIndex" :layout="'->, total, prev, pager, next, jumper'"
        :page-size="query.pageSize" :total="pageTotal" @current-change="handlePageChange"></el-pagination>
    </div>
  </div>
</template>

<style scoped>
/* upload button css */
.excel-upload-input {
  display: none;
  z-index: -9999;
}

.drop {
  border: 2px dashed #bbb;
  width: 600px;
  height: 120px;
  line-height: 120px;
  /* margin: 0 auto; */
  font-size: 24px;
  border-radius: 5px;
  text-align: center;
  color: #bbb;
  position: relative;
}

.note {
  align-self: center;
  text-align: center;
  position: fixed;
  right: 5rem;
  width: 20rem;
}

.container {
  padding: 20px 30px 0 30px;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 5px;
  margin: 20px 30px 0 30px;
}

.handle-box {
  margin-bottom: 20px;
}

.handle-input {
  width: 200px;
  display: inline-block;
}

.table {
  width: 100%;
  font-size: 14px;
}

.red {
  color: #ff0000;
}

.mr10 {
  margin-right: 10px;
}

.table-td-thumb {
  display: block;
  margin: auto;
  width: 40px;
  height: 40px;
}

.pagination {
  margin: 20px 0;
  text-align: right;
  padding: 0;
}
</style>
