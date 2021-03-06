<template>
  <el-main
    v-shortkey="shortsKey"
    @shortkey.native="keyAction"
  >
    <el-collapse v-model="activeAccordion" accordion>
      <el-collapse-item name="query-criteria">
        <template slot="title">
          Ver Histórico de Órdenes
        </template>
        <el-form
          v-if="!isEmptyValue(metadataList)"
          label-position="top"
          label-width="10px"
          @submit.native.prevent="notSubmitForm"
        >
          <template
            v-for="(field) in metadataList"
          >
            <field
              :key="field.columnName"
              :metadata-field="field"
            />
          </template>
        </el-form>
        <div
          v-else
          key="form-loading"
          v-loading="isEmptyValue(metadataList)"
          :element-loading-text="$t('notifications.loading')"
          element-loading-spinner="el-icon-loading"
          element-loading-background="rgba(255, 255, 255, 0.8)"
          class="loading-panel"
        />
      </el-collapse-item>
    </el-collapse>

    <el-table
      ref="orderTable"
      v-shortkey="shortsKey"
      v-loading="!tableOrder.isLoaded"
      :data="ordersList"
      border
      fit
      :highlight-current-row="highlightRow"
      :height="heightTable"
      @shortkey.native="keyAction"
      @current-change="handleCurrentChange"
    >
      <el-table-column
        prop="documentNo"
        label="Nro. Documento"
        width="130"
      />

      <el-table-column
        label="Estado"
        width="100"
      >
        <template slot-scope="scope">
          <el-tag
            :type="tagStatus(scope.row.documentStatus.value)"
          >
            {{ scope.row.documentStatus.name }}
          </el-tag>
        </template>
      </el-table-column>

      <el-table-column
        prop="salesRepresentative.name"
        label="Agente Comercial"
        min-width="170"
      />

      <el-table-column
        label="Socio de Negocio"
        min-width="150"
      >
        <template slot-scope="scope">
          {{ scope.row.businessPartner.name }}
        </template>
      </el-table-column>

      <el-table-column
        label="Fecha de Orden"
        width="135"
      >
        <template slot-scope="scope">
          {{ formatDate(scope.row.dateOrdered) }}
        </template>
      </el-table-column>
      <el-table-column
        label="Total General"
        align="right"
        width="120"
      >
        <template slot-scope="scope">
          {{ formatQuantity(scope.row.grandTotal) }}
        </template>
      </el-table-column>
    </el-table>

    <custom-pagination
      :total="tableOrder.recordCount"
      :current-page="tableOrder.pageNumber"
      :handle-change-page="handleChangePage"
    />
  </el-main>
</template>

<script>
import CustomPagination from '@/components/ADempiere/Pagination'
import fieldsListOrders from './fieldsListOrders.js'
import {
  createFieldFromDictionary
} from '@/utils/ADempiere/lookupFactory'
import {
  formatDate,
  formatQuantity
} from '@/utils/ADempiere/valueFormat.js'
import Field from '@/components/ADempiere/Field'

export default {
  name: 'OrdersList',
  components: {
    CustomPagination,
    Field
  },
  props: {
    metadata: {
      type: Object,
      default: () => {
        return {
          uuid: 'Orders-List',
          containerUuid: 'Orders-List'
        }
      }
    },
    showField: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      defaultMaxPagination: 50,
      fieldsList: fieldsListOrders,
      metadataList: [],
      isCustomForm: true,
      activeAccordion: 'query-criteria',
      timeOut: null
    }
  },
  computed: {
    heightTable() {
      if (this.isEmptyValue(this.activeAccordion)) {
        return 500
      }
      return 250
    },
    highlightRow() {
      if (!this.isEmptyValue(this.selectOrder)) {
        return true
      }
      return false
    },
    tableOrder() {
      return this.$store.getters.getListOrder
    },
    ordersList() {
      const order = this.tableOrder
      if (order && !this.isEmptyValue(order.ordersList)) {
        return order.ordersList
      }
      return []
    },
    selectOrder() {
      const action = this.$route.query.action
      const order = this.ordersList.find(item => item.uuid === action)
      if (!this.isEmptyValue(order)) {
        return order
      }
      return null
    },
    isReadyFromGetData() {
      const { isReload } = this.tableOrder
      return isReload
    },
    shortsKey() {
      return {
        closeOrdersList: ['esc'],
        refreshList: ['f5']
      }
    }
  },
  watch: {
    showField(value) {
      if (value && this.isEmptyValue(this.metadataList)) {
        this.setFieldsList()
      }
    }
  },
  created() {
    this.unsubscribe = this.subscribeChanges()
    if (this.isReadyFromGetData) {
      this.loadOrdersList()
    }
  },
  beforeDestroy() {
    this.unsubscribe()
  },
  methods: {
    formatDate,
    formatQuantity,
    createFieldFromDictionary,
    keyAction(event) {
      switch (event.srcKey) {
        case 'refreshList':
          this.loadOrdersList()
          break

        case 'closeOrdersList':
          this.$store.commit('showListOrders', false)
          break
      }
    },
    loadOrdersList() {
      let values = this.$store.getters.getValuesView({
        containerUuid: this.metadata.containerUuid
      })
      values = this.convertValuesToSend(values)
      const point = this.$store.getters.getPointOfSalesUuid
      if (!this.isEmptyValue(point)) {
        this.$store.dispatch('listOrdersFromServer', {
          ...values
        })
      }
    },
    handleChangePage(newPage) {
      this.$store.dispatch('setOrdersListPageNumber', newPage)
    },
    handleCurrentChange(row) {
      // close popover
      this.$store.commit('showListOrders', false)
      this.$store.dispatch('currentOrder', row)
      if (!this.isEmptyValue(row)) {
        this.$store.dispatch('deleteAllCollectBox')
        this.$router.push({
          params: {
            ...this.$route.params
          },
          query: {
            ...this.$route.query,
            action: row.uuid
          }
        }, () => {})
        const orderUuid = this.$route.query.action
        this.$store.dispatch('listPayments', { orderUuid })
      }
    },
    subscribeChanges() {
      return this.$store.subscribe((mutation, state) => {
        if (mutation.type === 'updateValueOfField' &&
          !mutation.payload.columnName.includes('DisplayColumn') &&
          !mutation.payload.columnName.includes('_UUID') &&
          mutation.payload.containerUuid === this.metadata.containerUuid) {
          clearTimeout(this.timeOut)
          this.timeOut = setTimeout(() => {
            this.loadOrdersList()
          }, 2000)
        }
      })
    },
    convertValuesToSend(values) {
      const valuesToSend = {}

      values.forEach(element => {
        const { value, columnName } = element

        if (this.isEmptyValue(value)) {
          return
        }

        switch (columnName) {
          case 'DocumentNo':
            valuesToSend['documentNo'] = value
            break
          case 'C_BPartner_ID_UUID':
            valuesToSend['businessPartnerUuid'] = value
            break
          case 'GrandTotal':
            valuesToSend['grandTotal'] = value
            break
          case 'OpenAmt':
            valuesToSend['openAmount'] = value
            break
          case 'IsPaid':
            valuesToSend['isPaid'] = value
            break
          case 'Processed':
            valuesToSend['isProcessed'] = value
            break
          case 'IsAisleSeller':
            valuesToSend['isAisleSeller'] = value
            break
          case 'IsInvoiced':
            valuesToSend['isInvoiced'] = value
            break
          case 'DateOrderedFrom':
            valuesToSend['dateOrderedFrom'] = value
            break
          case 'DateOrderedTo':
            valuesToSend['dateOrderedTo'] = value
            break
          case 'SalesRep_ID_UUID':
            valuesToSend['salesRepresentativeUuid'] = value
            break
        }
      })

      return valuesToSend
    },
    orderPrpcess(row) {
      const parametersList = [{
        columnName: 'C_Order_ID',
        value: row.id
      }]
      this.$store.dispatch('addParametersProcessPos', parametersList)
    },
    setFieldsList() {
      const list = []
      // Product Code
      this.fieldsList.forEach(element => {
        this.createFieldFromDictionary(element)
          .then(response => {
            const data = response
            list.push({
              ...data,
              containerUuid: 'Orders-List'
            })
          }).catch(error => {
            console.warn(`LookupFactory: Get Field From Server (State) - Error ${error.code}: ${error.message}.`)
          })
      })
      this.metadataList = list
    }
  }
}
</script>
