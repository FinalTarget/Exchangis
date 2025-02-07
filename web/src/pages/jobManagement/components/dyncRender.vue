<template>
  <div>
    <a-select
      v-if="type === 'OPTION'"
      v-model:value="value"
      ref="select"
      :options="checkOptions"
      @change="emitData(value)"
      :style="style"
      :placeholder="description"
    >
    </a-select>
    <template v-if="type === 'MAP'">
      <div style="margin-bottom: 5px" v-for="item in partitionArr">
        <a-input
          style="width: 30%"
          disabled
          v-model:value = item.label
        />
        <a-input
          v-if="item.type === 'INPUT'"
          v-model:value="item.value"
          @change="handleChange(item.value, item)"
          style="margin-left: 5px;width: 60%"
        />
        <a-select
          v-if="item.type === 'OPTION'"
          mode="tags"
          v-model:value="item.value"
          :maxTagCount="1"
          @change="handleChange(item.value, item)"
          :options="item.options"
          style="margin-left: 5px;width: 60%"
        />
      </div>
    </template>
    <template v-if="type === 'INPUT'">
      <a-input
        v-model:value="value"
        @change="emitData(value)"
        :style="style"
        :placeholder="description"
      />
      <span style="margin-left: 5px">{{unit}}</span>
    </template>

  </div>
</template>
<script>
import { defineComponent, h, toRaw, watch, computed, reactive, ref } from "vue";
import { getPartitionInfo } from "@/common/service";
import { message } from "ant-design-vue";

export default defineComponent({
  props: {
    param: Object,
    style: {
      type: Object,
      default: () => {
        return { width: '200px' }
      }
    },
    data: Object
  },
  emits: ["updateInfo"],
  setup(props, context) {
    let { type, field, value, unit, source, description} = props.param;
    value = ref(value)
    //let tmlName = field.split(".").pop();
    const newProps = computed(() => JSON.parse(JSON.stringify(props.param)))
    watch(newProps, (val, oldVal) => {
      value.value = val.value
      if (type === 'OPTION'){
        checkOptions.value = []
        val.values.map((item) => {
          checkOptions.value.push({
            value: item,
            label: item
          })
        })
      }
      /*if (type === 'MAP') {
        value.value = value.value || {}
        if (val.source === oldVal.source) {
          partitionArr.value.forEach(partition => {
            if (partition.type === 'OPTION') {
              partition.value = value.value[partition.label] ? [value.value[partition.label]] : []
            } else {
              partition.value = value.value[partition.label] ? value.value[partition.label] : partition.defaultValue || ''
            }
          })
        }
      }*/
    })
    const newData = computed(() => JSON.parse(JSON.stringify(props.data || {})))
    watch(newData, (val, oldVal) => {
      if (type === 'MAP') {
        if (val.id != oldVal.id || val.db !== oldVal.db || val.table !== oldVal.table) {
          _buildMap()
        }
      }
    })
    let checkOptions = ref([])
    if (type === 'OPTION'){
      checkOptions.value = []
      props.param.values.map((item) => {
        checkOptions.value.push({
          value: item,
          label: item
        })
      })
    }
    let partitionArr = ref([])
    const _buildMap = function () {
      partitionArr.value = []
      let url
      const source_reg = new RegExp('^http');
      if (source_reg.test(source.value)) {
        url = source.split('?')[0]
      } else {
        url = window.location.origin + source.split('?')[0]
      }
      getPartitionInfo({
        source: url,
        dataSourceId: props.data.id,
        database: props.data.db,
        table: props.data.table
      })
        .then(res => {
          for (let i in res.render) {
            if (typeof(res.render[i]) === 'string') {
              partitionArr.value.push({
                type: 'INPUT',
                label: i,
                value: value.value && value.value[i] ? value.value[i] : res.render[i],
                defaultValue: res.render[i]
              })
            } else {
              let checkOptions = []
              res.render[i].map((item) => {
                checkOptions.push({
                  value: item,
                  label: item
                })
              })
              partitionArr.value.push({
                type: 'OPTION',
                label: i,
                options: checkOptions,
                value: value.value && value.value[i] ? [value.value[i]] : []
              })
            }
          }
        })
        .catch(err => {
          message.error("获取分区信息失败");
        })
    }
    if (type === 'MAP') {
      _buildMap()
    }
    const emitData = (value) => {
      let res = toRaw(props.param)
      res.value = value
      context.emit("updateInfo", res)
    }

    // 处理map变化
    const handleChange = (value, item) => {
      if (item.type === 'OPTION') {
        if (value && value.length) {
          item.value = [value.pop()]
        }
      }
      let res = toRaw(props.param)
      if (!res.value) {
        res.value = {}
      }
      partitionArr.value.map(part => {
        if (part.type === 'OPTION') {
          if (part.value && part.value.length) {
            res.value[part.label] = part.value.join()
          } else {
            delete res.value[part.label]
          }
        } else {
          if (part.value) {
            res.value[part.label] = part.value
          } else {
            delete res.value[part.label]
          }
        }
      })
      context.emit("updateInfo", res)
    }

    return {
      checkOptions,
      type,
      value,
      emitData,
      unit,
      source,
      partitionArr,
      description,
      handleChange
    }
  }
});
</script>

<style scoped>
.custom_input {
  width: 226px;
  height: 25px;
}

.custom_select {
  width: 226px;
  height: 25px;
}
.custom_span_unit {
  width: 30px;
  margin-left: 8px;
  white-space: nowrap;
  display: inline-block;
  text-align: start;
}
</style>
