<template>
    <div>
        <el-input v-model="insert" :rows="3" type="textarea" @input="changeVal" placeholder="请输入sql, 如insert into COMPANIES (COMPANY, TYPE, NAME, ADDRESS1) values ('HL00R', 'V', '南京汇润霖信息科技有限公司(测试用）', '南京市秦淮区太平南路465号');" />
        <el-container>
            <el-main class="main">
                <div v-for="(item,index) in input1Arr" :key="index">
                    <el-input :name="Object.keys(item)[0]" class="w-50 m-2 main_input" :placeholder="`${index+1}.${Object.keys(item)[0]}`" disabled>
                    </el-input>
                    <el-input v-model="item[Object.keys(item)[0]]" class="w-50 m-2 main_input" :placeholder="Object.values(item)[0]" @input="changeExcel(item)">
                        <!-- <template #append>
                            <el-button :icon="Check" type="success" plain round/>
                        </template> -->
                        <template #suffix>
                            <el-icon class="el-input__icon" :size="20" :color="item.color=='red'?'red':'grey'" @click="toExcel(item)">
                                <EditPen />
                            </el-icon>
                        </template>
                    </el-input>
                </div>
            </el-main>
        </el-container>
        <el-button type="primary" round @click="generateSql()">生成sql</el-button>
        <el-button type="danger" round @click="generateSqlBatch">批量生成sql</el-button>
        <component :is="ExcelXls" ref="Excel" class="ExcelXls"></component>

        <!-- 抽屉 -->
        <el-drawer v-model="drawer2" :direction="direction" size='80%' custom-class="myDrawer">
            <template #title>
                <div style="height:450px;overflow-y: scroll;">
                    <h4 v-for="(item,index) in sqlBatch" :key="index">
                        {{item}}
                    </h4>
                </div>
            </template>
            <template #footer>
                <div style="flex: auto">
                    <el-button @click="cancelClick">cancel</el-button>
                    <el-button type="primary" @click="confirmClick">confirm</el-button>
                </div>
            </template>
        </el-drawer>
        <div>
            <p style="font-size: 5px;margin-left: 10px;color: grey;font-style:italic;">* 支持取值模式,取值模式必须由双++,或者双~~包裹(+D3+,~D3~),D3即对应单元格的地址</p>
            <p style="font-size: 5px;margin-left: 10px;color: grey;font-style:italic;">* 取值模式下,<span style="color: red">~D3~</span> 取出的值不含单引号,<span style="color: red">+D3+</span> 取出的值包含单引号</p>
            <p style="font-size: 5px;margin-left: 10px;color: grey;font-style:italic;">* 取值模式下,目前仅支持单张Excel表的Sheet1,多个Sheet页是不支持的</p>
            <p style="font-size: 5px;margin-left: 10px;color: grey;font-style:italic;">* 目前仅支持 <span style="color: red">insert</span> 语句</p>
        </div>
    </div>
</template>

<script>
export default {
    name: 'HelloWorld',
    data() {
        return {
        };
    },
    methods: {},
    mounted() { },
}
</script>
<script setup>
import { ref, computed } from 'vue'
import { nextTick } from 'process'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Check, EditPen } from "@element-plus/icons-vue"
import ExcelXls from './ExcelXls.vue'


let input1Arr = ref([{ Field1: 'Value1' }, { Field2: 'Value2' }, { Field3: 'Value3' }])
let input2Arr = ref([{ Field1: 'Value1' }, { Field2: 'Value2' }, { Field3: 'Value3' }])
const insert = ref('')
const sql = ref('')
const sqlBatch = ref([])

const drawer2 = ref(false)
const direction = ref('rtl')

const Excel = ref(null)

//检测输入sql,并拆分处理
function changeVal() {
    const reg = /[) values (|)values (|) values(|)values(]/g
    if (reg.test(insert.value)) {
        //获取属性名字
        let field = insert.value.substring(insert.value.indexOf('(') + 1, insert.value.indexOf(')'))
        //获取属性对应的值
        let sub1 = insert.value.substring(insert.value.indexOf('values') + 6)
        let value = sub1.substring(sub1.indexOf('(') + 1, sub1.lastIndexOf(')'))
        //如果某个value内包含了多个,做一个简单的转换,---暂时不做
        //split后的值可能会包含to_date('06-10-2022 20:50:26', 'dd-mm-yyyy hh24:mi:ss')这样的值被误拆分,需要处理
        let valueArr = value.split(',')
        let valueArr1 = []
        let removeArr = []
        valueArr.forEach((item, index) => {
            if (removeArr.indexOf(index) != -1) return 
            // if(index==17) debugger // eslint-disable-line
            const regs = /^'.*[^']$/g //以'开头,不以'结束
            const rege = /.*'$/g //不以'开头,以'结束
            const regx = /to_date|to_char/g
            if (regx.test(item.trim())) {
                valueArr1.push(item.trim() + ',' + valueArr[index + 1].trim())
                removeArr.push(index+1)
            } else if (regs.test(item.trim())) {
                let ind = index
                let ele = item
                while(!rege.test(ele)){
                        rege.lastIndex = 0
                        ind = ind+1
                        ele = valueArr[ind]
                        removeArr.push(ind)
                        item = item.trim()+ele
                }
                valueArr1.push(item)
            }else {
                valueArr1.push(item.trim())
            }
        })

        //对field和value做键值对匹配
        let fieldArr = field.split(',')
        input1Arr.value = fieldArr.map((item, index) => {
            let obj = {}
            obj[item] = valueArr1[index]
            return obj
        })
    }
}
//批量生成sql
function generateSqlBatch() {
    // debugger // eslint-disable-line
    //先对从哪行开始计数做一个粗略计算
    const first = input1Arr.value.find((item, index) => {
        return !!item.isExcel
    })
    if (first !== undefined) {
        let reg = /\+.*\+/g
        let reg1 = /\d+/g
        const v1 = reg.exec(Object.values(first)[0])
        const v2 = v1 === null ? reg1.exec(Object.values(first)[0])[0] : reg1.exec(v1[0])[0]
        ElMessageBox.confirm(
            '检测到您是想从第' + v2 + '行开始',
            'Warning',
            {
                confirmButtonText: '是',
                cancelButtonText: '否',
                type: 'warning',
            }
        )
            .then(() => {
                Excel.value.startRow = v2
                try {
                    generateSqlBatchs()
                } catch (e) {
                    ElMessage({
                        message: e,
                        type: 'warning',
                    })
                }
            })
            .catch(() => {
                ElMessage({
                    message: '暂时未开发~~~~,',
                    type: 'warning',
                })
            })
    } else {
        ElMessage({
            type: 'warning',
            message: '未指定取值模式,无法批量生成sql',
        })
    }
    
}
//批量生成sql核心方法
function generateSqlBatchs(){
    // debugger // eslint-disable-line
    //先对input1Arr进行copy ,目的是为防止回写值到页面
    //input1Arr具有响应式,按如下处理,得到失去响应式的对象
    const origin= input1Arr.value.map((item) => {
        let obj = {}
        Object.keys(item).forEach((ele,inx) => {
            obj[ele] = item[ele]
        })
        return obj 
    })
    //对input2Arr重新写值并循环调用generateSql
    const startRow = Excel.value.startRow
    const maxRow = Excel.value.maxRow
    if (maxRow==0) ElMessage({ type: 'warning', message: '没检测到文件,要不上传个文件试试?', })
    for (let i = startRow; i <= parseInt(maxRow); i++) {
        const ina = origin.map((item, index) => {
            // debugger // eslint-disable-line
            if (item.isExcel) {
                let reg = /\+.*\+/g
                let reg1 = /\d+/g
                const v1 = reg.exec(Object.values(item)[0])
                const ev = v1 === null ? reg1.exec(Object.values(item)[0]) : reg1.exec(v1[0])
                if (ev === null) {
                    ElMessage({ type: 'warning', message: Object.values(item)[0]+' 值有误', })
                    return item
                } else {
                    let vnumber = i == startRow ? parseInt(ev[0]) : parseInt(ev[0]) + 1
                    reg.lastIndex = 0
                    reg1.lastIndex = 0
                    item[Object.keys(item)[0]] = v1===null ? Object.values(item)[0].replace(ev[0],vnumber) : Object.values(item)[0].replace(v1[0],v1[0].replace(ev[0],vnumber))
                    // console.log(item,'item');
                    return item
                }
                
            } else {
                return item
            } 
        })
        //执行generateSql
        generateSql(ina)
    }
}
//生成最终sql
function generateSql(ina) {
    //获取主表名称
    let ownertable = insert.value.substring(insert.value.indexOf("insert into") + 11, insert.value.indexOf('('))
    let sql_Field = ''
    let sql_Value = ''
    ina = ina!==undefined ? ref(ina) : input1Arr
    ina.value.forEach((item, index) => {
        const reg = /\+.*\+|~.*~/g
        sql_Field += index != ina.value.length - 1 ? Object.keys(item)[0] + ',' : Object.keys(item)[0]
        //处理了如何获取单元格的值
        if (!!item.isExcel) {
            const matchArr = Object.values(item)[0].match(reg)
            if (matchArr === null) {
                const value = Excel.value.getValue(Object.values(item)[0]).v
                sql_Value += index != ina.value.length - 1 ? value + ',' : value
            } else {
                let realvalue
                matchArr.forEach((match, index) => {
                    // debugger // eslint-disable-line
                    const replaceValue = match.replaceAll(/\+|~/g, "").trim()
                    //当查找的单元格实际是空值的时候,则使用字符串''替代返回的值
                    const excelValue = Excel.value.getValue(replaceValue)===undefined ? '' : Excel.value.getValue(replaceValue).v
                    realvalue = Object.values(item)[0].replace(replaceValue, excelValue)
                })

                const flag = reg.test(realvalue)
                reg.lastIndex = 0 //test或者exec执行完成后lastIndex会变,需要重置为0,下一次执行其他函数才会匹配到结果
                const realreg = /\+.*\+/g.exec(realvalue)
                const flag1 = realreg !==null ? realreg.index ==0 : false
                // const flag1 = /\+.*\+/g.exec(realvalue).index == 0
                const vflag = flag && flag1
                // debugger // eslint-disable-line
                switch (flag1) {
                    case true:
                        realvalue = realvalue.replaceAll(/\+/g, "").trim()
                        break;
                    case false:
                        if (/~.*~/g.test(realvalue)) {
                            realvalue = realvalue.replaceAll(/~/g, "").trim()
                        } else {
                            realvalue = realvalue.replaceAll(/\+/g, "'").trim()
                        }
                        break;
                    default:
                        realvalue = realvalue.replaceAll(/\+|~/g, "").trim()
                }

                switch (vflag) {
                    case true:
                        sql_Value += index != ina.value.length - 1 ? "'" + realvalue + "'" + ',' : "'" + realvalue + "'"
                        break;
                    case false:
                        sql_Value += index != ina.value.length - 1 ? realvalue + ',' : realvalue
                        break;
                    default:
                        sql_Value += index != ina.value.length - 1 ? "'" + realvalue + "'" + ',' : "'" + realvalue + "'"
                }
            }
        } else {
            sql_Value += index != ina.value.length - 1 ? Object.values(item)[0] + ',' : Object.values(item)[0]
        }
    })
    sql_Field = ' (' + sql_Field + ') '
    sql_Value = ' (' + sql_Value + ') '
    sql.value = "insert into " + ownertable.trim() + sql_Field + ' values ' + sql_Value + ';'
    // console.log(sql);
    sqlBatch.value.push(sql.value)
    drawer2.value = true
}
//将键值对进行标记,需要获取excel中对应的值则勾选
function toExcel(item) {
    // debugger // eslint-disable-line
    item.color = item.color == 'red' ? 'grey' : 'red'
    /* input1Arr.value.forEach((ele, inx) => {
        if (Object.keys(ele)[0] == Object.keys(item)[0]) {
            if (item.color == 'red') {
                input1Arr.value[inx].isExcel = true
            }
            if (item.color == 'grey') {
                input1Arr.value[inx].isExcel = false
            }
        }
    }) */
    item.isExcel = item.color == 'red' ? true :false
    // console.log(input1Arr.value);
    // input1Arr.value = [...input1Arr.value,{...item,isExcel:true}]
}
//自动标记是否是isExcel
function changeExcel(item){
    let reg = /\+.*\+|~.*~/g
    if (reg.test(Object.values(item)[0])) {
        item.color = 'red'
        item.isExcel = true
    } else {
        item.color = 'grey'
        item.isExcel = false
    }
}
function cancelClick() {
    drawer2.value = false
    sql.value = ''
    sqlBatch.value = []
}
function confirmClick() {
    ElMessageBox.confirm(`Are you confirm to copy ?`)
        .then(() => {
            drawer2.value = false
        })
        .catch(() => {
            // catch error
        })
}
</script>

<style scoped>
.main {
    height: 300px;
    border: 0.5px rgb(209, 207, 207) solid;
    border-radius: 5px;
    margin-top: 10px;
    margin-bottom: 10px;
    padding: 0 5px;
}
.main_input {
    margin: 5px 0;
    width: 48%;
}
.main_input:first-child {
    margin-right: 4%;
}
.ExcelXls{
    display: inline-block;
    margin: 0 10px;
}
</style>