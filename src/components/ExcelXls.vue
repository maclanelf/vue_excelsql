<template>
    <div>
        <el-upload accept=".xlsx,.xls" :before-upload="upload" action="" :limit="1">
            <template #trigger>
                <el-button type="danger" round>select file</el-button>
                <div class="el-upload__tip text-red bottomRow">
                    {{startRow}} / {{maxRow}}
                </div>
            </template>
            <!-- <template #tip>
                <div class="el-upload__tip text-red">
                    {{startRow}} / {{maxRow}}
                </div>
            </template> -->
        </el-upload>
    </div>
</template>
<script>
export default {
    name: 'ExcelXls',
    components: {},
    props: [''],
    data() {
        return {

        };
    }
}
</script>
<script setup>
import { ref, onMounted, defineExpose } from "vue";
import { UploadFilled, Edit, Upload } from "@element-plus/icons-vue"
import { ElMessage } from 'element-plus'
import { read, readFile, utils, writeFileXLSX } from "xlsx";

const worksheet = ref(null);
const startRow = ref(2)
const maxRow = ref(0)

//上传文件并将excel的worksheet对象进行存储,以便取值
const upload = (file) => {
    const fileReader = new FileReader()
    fileReader.onload = (ev) => {
        try {
            const data = ev.target.result
            const workbook = read(data, {
                type: "binary"
            })
            // 取第一张表
            const wsname = workbook.SheetNames[0]
            // 生成json表格内容
            worksheet.value = workbook.Sheets[wsname]
            const ws = utils.sheet_to_json(workbook.Sheets[wsname])
            // console.log(worksheet.value["B4"]);
            //获取excel单表的最大有效值范围,A1:H457
            const valid = worksheet.value["!ref"]
            const valid1 = valid.substring(valid.indexOf(":")+1)
            maxRow.value = /\d+/g.exec(valid1)[0]
        } catch (e) {
            return false
        }
    }
    fileReader.readAsBinaryString(file)
    // 阻止ElUpload默认上传
    return false
}
//获取单元格的值
const getValue = (address) => {
    // debugger // eslint-disable-line
    if (worksheet.value === null || worksheet.value === undefined) {
        ElMessage({
            message: '没检测到文件,要不上传个文件试试?',
            type: 'warning',
        })
        return 
    }
    if (worksheet.value[address] === null || worksheet.value[address] === undefined) {
        ElMessage({
            message: '未找到单元格 '+address+' 的值,请确认是否填写正确',
            type: 'warning',
        })
    }
    return worksheet.value[address]
}
//setup标签中需要暴露出去,父组件才能调用,setup()中不用
defineExpose({ getValue ,startRow,maxRow})
</script>
<style scoped>
.bottomRow{
    margin-left: 10px;
}
</style>