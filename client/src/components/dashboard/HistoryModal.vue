<template>
 <VModal :showModal="showModal" @closeModal="closeModal" v-bind="{ 'background-color': '#DCE4F2', 'max-height': '45rem', 'max-width': '77rem' }">
    <template v-slot:header v-if="selectedHistoryRow">
      {{ selectedHistoryRow.message }}
    </template>
      
    <template v-slot:body v-if="selectedHistoryRow">
      <template v-if="actionType === 'delete'">
        <div v-if="selectedHistoryRow.entity.includes('empty')">
          <!-- TODO: add this msg to `additional_info` -->
          <p v-if="getEmptyEntityName === 'document'">
            This document does not exist anymore. <br>
          </p>
          <p>
            There are no more {{ getEmptyEntityName }}s
          </p>
        </div>

        <div class="c-table">
            <VTableSimple 
              :columns="getPropertiesOfPropArr(modalData)"
            >
              <template v-slot:tbody>
                
                <tr v-for="(item, index) in modalData" :key="index">
                  <td v-for="columnName in $options.currentEntityColumns" :key="index + columnName">
                    {{ item[columnName] }}
                  </td>
                </tr>

              </template>
            </VTableSimple>
          </div>
      </template>

      <template v-else-if="actionType === 'update'">
        <template v-for="(fromTo, id) in modalData">
          <p :key="id"><b>{{ id }}</b></p>

          <p :key="id + 'from'">From</p>
          <div :key="id + 'table'" class="c-table">
            <VTableSimple 
              :columns="getPropertiesOfNestedObj(fromTo)"
            >
              <template v-slot:tbody>
                <tr>
                  <td v-for="column in $options.currentEntityColumns" :key="id + column">
                    {{ fromTo.from[column] }}
                  </td>
                </tr> 
              </template>
            </VTableSimple>
          </div>

          <p :key="id + 'to'">To</p>
          <div :key="id + 'table' + 'to'" class="c-table">
            <VTableSimple 
              :columns="$options.currentEntityColumns"
            >
              <template v-slot:tbody>
                <tr>
                  <td v-for="column in $options.currentEntityColumns" :key="column + id">
                    {{ fromTo.to[column] }}
                  </td>
                </tr>
              </template>
            </VTableSimple>
          </div>
        </template>
      </template>

      <template v-else-if="actionType === 'insert'">
        <div class="c-table">
          <VTableSimple
            :columns="$options.currentEntityColumns"
          >
            <template v-slot:tbody>
              <!-- Using `index` as a key because this modal is readonly -->
              <tr v-for="(createdItem, index) in modalData" :key="index">
                <td v-for="column in $options.currentEntityColumns" :key="column + index">
                  {{ createdItem[column] }}
                </td>
              </tr> 
            </template>
          </VTableSimple>
        </div>

        <template v-if="additionalInfo">
          <template v-for="(additionalItemObj, title, index) in additionalInfo">
            <p :key="index + 'title'">{{ title }}</p>

            <div :key="index + title" class="c-table">
                <VTableSimple
                  :columns="getPropertiesOfPropArr(additionalInfo[title])"
                >
                <template v-slot:tbody>
                  <tr v-for="(additionalItem, index) in additionalItemObj" :key="index">
                    <td v-for="column in $options.currentEntityColumns" :key="column + index + title">
                      {{ additionalItem[column] }}
                    </td>
                  </tr> 
                </template>
              </VTableSimple>
            </div>
          </template>
        </template>
      </template>

      <div align="right">
        {{ formatDate(selectedHistoryRow.inserted_date) }}
      </div>
    </template>
  </VModal>
</template>

<script>
import VModal from '../VModal';
import VTableSimple from '../VTableSimple';

import { mapState } from 'vuex'

import uuidv1 from 'uuid/v1'

import modalMixin from '../../mixins/modalMixin';

import { separateValues, formatDate, getRidOfObjProp, getPropertiesOfNestedObj } from '../../utils/index';

import historySharedData from '../../observables/history';

export default {
    name: 'history-modal',

    components: { VModal, VTableSimple },

    /**
     * We don't want this variable to be reactive, as we might get into infinite loops
     * when mutating this variable
     */
    currentEntityColumns: null,

    data: () => ({
      actionType: null,
      modalData: null,
      /**
       * For example, when creating a product, we need to keep tract of
       * the `created provider` and the `created products`
       */
      additionalInfo: null
    }),

    mixins: [modalMixin],

    watch: {
        showModal (shouldDisplayModal) {
            shouldDisplayModal
                ?
                window.addEventListener("keyup", this.modalHandler) :
                window.removeEventListener("keyup", this.modalHandler)
        },

        selectedHistoryRow (row) {
          if (!row)
            return;

          console.log(row)
          this.actionType = row.action_type;

          switch (this.actionType) {
            case 'delete': {
              this.modalData = (JSON.parse(row.prev_state));
              break;
            }
            
            case 'update': {
              this.modalData = JSON.parse(row.current_state);
              break;
            }

            case 'insert': {
              this.modalData = JSON.parse(row.current_state);
              row.additional_info && (this.additionalInfo = JSON.parse(row.additional_info))
              this.$options.currentEntityColumns = Object.keys(this.modalData[0]);
              
              break;
            }
          }
        }
    },

    computed: {
        // Decided to name this way because the `modalMixin` already contains a property `showDetails`
        showModal () { return historySharedData.showModal },

        getEmptyEntityName () {
          return this.selectedHistoryRow.entity.slice(0, this.selectedHistoryRow.entity.indexOf('/'))
        },

        selectedHistoryRow () { return historySharedData.selectedHistoryRow },

        getHistoryProductNames () {
            return this.selectedHistoryRow.additional_info.split('|')
        },

        ...mapState('dashboard', ['documentIds'])
    },

    methods: {
        getPropertiesOfNestedObj (obj) {

          this.$options.currentEntityColumns = getPropertiesOfNestedObj(obj);

          return this.$options.currentEntityColumns;
        },

        getPropertiesOfPropArr (arr) {
          this.$options.currentEntityColumns = Object.keys(arr[0]);

          return this.$options.currentEntityColumns;
        },

        formatDate (date) { return formatDate(date) },

        getRidOfObjProp (obj, prop) { return getRidOfObjProp(obj, prop) },

        closeModal () {
            historySharedData.showModal = false
            historySharedData.selectedHistoryRow = null
        },
    },
}
</script>

<style lang="scss">
    $main-blue: #394263;

    .c-table {
        display: flex;
        justify-content: center;
        margin-bottom: 1rem;
        margin-top: 1rem;
    }

    .redirect-link {
    color: darken($color: $main-blue, $amount: 15%);
    font-style: italic;

    &:hover {
      font-weight: bold;
    }
  }
</style>
