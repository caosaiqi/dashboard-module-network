<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">新建</div>
    <div slot="body">
      <a-form
        :form="form.fc">
        <a-form-item label="专有网络" v-bind="formItemLayout" :validateStatus="vpcValidateStatus" :help="vpcHelp" :required="true">
          <a-row :gutter="8">
            <a-col :span="12">
              <a-select
                v-decorator="decorators.area"
                @change="handleAreaChange"
                placeholder="区域">
                <a-select-option v-for="item in areas" :value="item.value" :key="item.value">
                  {{item.label}}
                </a-select-option>
              </a-select>
            </a-col>
            <a-col :span="12">
              <a-select
                  v-decorator="decorators.vpc"
                  placeholder="专有网络">
                <a-select-option v-for="item in vpcs" :value="item.value" :key="item.value">
                  {{item.label}}
                </a-select-option>
              </a-select>
            </a-col>
          </a-row>
        </a-form-item>
        <a-form-item label="可用区" v-bind="formItemLayout">
          <a-select
              v-decorator="decorators.zone"
              placeholder="可用区">
            <a-select-option v-for="item in zones" :value="item.value" :key="item.value">
              {{item.label}}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item label="名称" v-bind="formItemLayout">
          <a-input v-decorator="decorators.name" placeholder="字母开头，数字和字母大小写组合，长度为2-9个字符，不含'.'，'_'，'@'" />
        </a-form-item>
        <a-form-item label="带宽" v-bind="formItemLayout">
          <a-select
              v-decorator="decorators.bandwidth"
              placeholder="网络带宽">
            <a-select-option v-for="item in bandwidthOptions" :value="item.value" :key="item.value">
              {{item.label}}
            </a-select-option>
          </a-select>
        </a-form-item>
      </a-form>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import { BAND_WIDTH_OPTION } from '../../../constants'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'
export default {
  name: 'WireCreateDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    // 自定义校验名称重复
    const validateName = (rule, value, callback) => {
      // 编辑模式下不需要校验重复
      if (this.$route.query.id) {
        callback()
      }
      let wiresManager = new this.$Manager('wires')
      wiresManager.list({ params: {
        filter: 'name.equals(' + value + ')',
      } })
        .then((res) => {
          let data = res.data.data
          if (data.length === 0) {
            callback()
          }
          callback(new Error('该名称已经存在'))
        })
    }
    return {
      loading: false,
      vpcValidateStatus: '',
      vpcHelp: '',
      form: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        area: [
          'area',
          {
            rules: [
              { required: true, message: '请选择区域' },
            ],
          },
        ],
        vpc: [
          'vpc',
          {
            rules: [
              { required: true, message: '请选择专有网络' },
            ],
          },
        ],
        zone: [
          'zone',
          {
            rules: [
              { required: true, message: '可用区' },
            ],
          },
        ],
        name: [
          'name',
          {
            initialValue: '',
            validateTrigger: ['change', 'blur'],
            validateFirst: true,
            rules: [
              { required: true, message: '请输入名称' },
              { validator: this.$validate('wiresName') },
              { validator: validateName, trigger: ['blur'] },
            ],
          },
        ],
        bandwidth: [
          'bandwidth',
          {
            rules: [
              { required: true, message: '请选择带宽' },
            ],
          },
        ],
      },
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
      areas: [],
      vpcs: [],
      zones: [],
      bandwidthOptions: BAND_WIDTH_OPTION,
    }
  },
  created () {
    this.fetchAreas()
    this.fetchZones()
  },
  methods: {
    fetchAreas () {
      let manager = new this.$Manager('cloudregions')
      manager.list({ params: {
        is_on_premise: true,
      } }).then((res) => {
        this.areas = res.data.data.map((item) => {
          return {
            label: item.name,
            value: item.id,
          }
        })
      })
    },
    fetchVpcs (area) {
      let manager = new this.$Manager('vpcs')
      manager.list({ params: {
        cloudregion_id: area,
      } }).then((res) => {
        this.vpcs = res.data.data.map((item) => {
          return {
            label: item.name,
            value: item.id,
          }
        })
      })
    },
    fetchZones () {
      let manager = new this.$Manager('zones')
      manager.list({ params: {
        is_on_premise: true,
      } }).then((res) => {
        this.loadZoneOptions(res.data.data)
      })
    },
    loadZoneOptions (data) {
      const results = []
      for (let i = 0; i < data.length; i++) {
        let zone = data[i]
        let name = zone.name
        if (zone.name_cn) {
          name += '(' + zone.name_cn + ')'
        }
        results.push({ label: name, value: zone.id })
      }
      this.zones = results
    },
    handleAreaChange (e) {
      this.fetchVpcs(e)
    },
    doCreate (data) {
      const params = {
        name: data.name,
        zone_id: data.zone,
        vpc_id: data.vpc,
        bandwidth: data.bandwidth,
      }
      return this.params.list.onManager('create', {
        managerArgs: {
          data: params,
        },
      })
    },
    async handleConfirm () {
      this.loading = true
      try {
        if (!this.form.fc.getFieldValue('area')) {
          this.vpcValidateStatus = 'error'
          this.vpcHelp = '请选择区域'
          this.loading = false
          return
        } else if (!this.form.fc.getFieldValue('vpc')) {
          this.vpcValidateStatus = 'error'
          this.vpcHelp = '请选择专有网络'
          this.loading = false
          return
        }
        this.vpcValidateStatus = ''
        this.vpcHelp = ''
        const values = await this.form.fc.validateFields()
        await this.doCreate(values)
        this.loading = false
        this.cancelDialog()
      } catch (error) {
        this.loading = false
      }
    },
  },
}
</script>
