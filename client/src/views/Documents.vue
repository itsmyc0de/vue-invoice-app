<template>
    <div v-if="isEverythingLoaded">
        <VContent 
            :entityName="entity"
            :disableButton="
                errorMessage !== 'Documents' 
                    && errorMessage !== null 
                    || vat['food_vat'] === null 
                    || vat['non_food_vat'] === null
                "
            :disableCreateButton="disableCreateButton"
            @insertCreatedItems="onInsertCreatedItems"
            :shouldDisplayConfirmCancelButtons="shouldDisplayConfirmCancelButtons"
            @confirmChanges="onConfirmChanges"
            @cancelChanges="onCancelChanges"
            >
            <template v-slot:existingItems>
                <VTableRead 
                    v-if="!containsErrors && !!items"
                    :fields="['id', ...readColumns, 'actions']" 
                    :items="items"
                    showDelete
                    @showInfo="showInfo($event)"
                    @deleteRow="prepareRowForDeletion($event)"
                />

                <div class="no-items" v-else-if="vat['food_vat'] === null || vat['non_food_vat'] === null">
                    Please make sure you specify the VAT!
                </div>

                <div v-else class="no-items">
                    There are no {{ errorMessage }}
                </div>
            </template>

            <template v-slot:createItems>
                <div class="l-flex">
                    <VButton @click="addRow" :disabled="!productsAsList.length">
                        <font-awesome-icon class="is-base-icon" icon="plus-circle" />
                    </VButton>
                    <VSelect @addProvider="$store.commit('SET_PROVIDER', $event)" class="c-select" :items="providers" />
                    <VInput @blur.native="$store.commit('SET_PROVIDER_INVOICE_NR', $event.target.value)" placeholder="Invoice Nr" class="c-input" />
                    <VVat />
                </div>
                
                <VTableCreate 
                    @deleteRow="deleteRowInstantly($event)" 
                    :fields="createColumns" 
                    :items="createdItems"
                    @addField="addField($event)"
                    @tableCreateReady="onTableCreateReady"
                    :listItems="productsAsList"
                />
            </template>
        </VContent>

        <VModal :showModal="showDetails" :isAboutToDelete="isAboutToDelete" @closeModal="closeModal">
            <template v-slot:header>
                <span>{{ modalTitle }}</span>
            </template>
            <template v-slot:body>
                <div
                    v-for="keyVal in Object.entries(selectedItem)"
                    :key="keyVal[0]"
                    class="modal-body__row"
                >
                    <div class="modal-body__prop"><span>{{ keyVal[0] }}</span></div>
                    <div class="modal-body__arrow"><font-awesome-icon icon="arrow-right" /></div>
                    <div class="modal-body__value">
                        <span>{{ keyVal[1] }}</span>
                    </div>
                </div>

                <div class="c-modal-buttons">
                    <button class="c-modal-buttons__button c-modal-buttons--yes" @click="confirmDelete">Yes</button>
                    <button class="c-modal-buttons__button c-modal-buttons--no" @click="cancelDelete">No</button>
                </div>
            </template>
        </VModal>

    </div>
</template>

<script>
import VContent from '../components/VContent';
import VModal from '../components/VModal';
import VTableCreate from '../components/VTableCreate';
import VTableRead from '../components/VTableRead';
import VSelect from '../components/VSelect';
import VInput from '../components/VInput';
import VVat from '../components/VVat';
import VButton from '../components/VButton';

import modalMixin from '../mixins/modalMixin';
import commonMixin from '../mixins/commonMixin';
import documentMixin from '../mixins/documentMixin';

const entityName = 'document';
const providerEntity = 'provider';
const productEntity = 'product';

import { createNamespacedHelpers } from 'vuex';
import * as common from '@/store/modules/common';

const { mapState, mapActions, mapGetters } = createNamespacedHelpers(entityName);
const { mapGetters: mapGettersProvider } = createNamespacedHelpers(providerEntity);
const { mapGetters: mapGettersProduct } = createNamespacedHelpers(productEntity);

import { capitalize } from '@/utils/';

export default {
    name: 'documents',

    components: { VContent, VModal, VTableCreate, VTableRead, VSelect, VInput, VVat, VButton },

    mixins: [modalMixin, commonMixin, documentMixin],

    data: () => ({
        errorMessage: null,
        isEverythingLoaded: false,
        entity: entityName,
        createdItemsObservee: 'document/getCreatedItemsAsArr'
    }),

    methods: {
        ...mapActions([
            'deleteCreatedItem', 'addFieldValue', 
            'updateItem', 'addCreatedItem', 'resetCreatedItems',
            'insertCreatedItems', 'deleteItem',
            'sendModifications',
            'resetCUDItems',
            'sendHistoryData',
            'resetChanges',
        ]),

        showInfo ({ id }) {
            this.$router.push({ name: 'documentEditOne', params: { id } });
        },

        prepareCreatedItemsForHistory () {
            console.log(this.createdItems)
            const documentProducts = {};
            const { id: providerId, invoiceNr, name, URC } = this.$store.state.selectedProvider;

            const createdProductsForHistory = this.createdItems.map(createdItem => {
                const { id: randomProductId, product_name: productObj, ...itemWithoutProductObj } = createdItem;
                const { id, ...productObjWithoutId } = productObj;

                return {
                    ...itemWithoutProductObj,
                    ...productObjWithoutId
                };
            });

            return {
                current_state: JSON.stringify([{ 
                    id: providerId, 
                    provider_name: name, 
                    provider_URC: URC, 
                }]),
                additional_info: JSON.stringify({products: createdProductsForHistory})
            };
        },

        async onInsertCreatedItems () {
            const response = await this.insertCreatedItems();
            this.openModalBox(capitalize(response.message));

            this.sendCreatedHistoryData(this.prepareCreatedItemsForHistory());

            this.refetchMainOverview();
            await this.fetchItems();
        },

        async onConfirmChanges () {
            console.log('confirm')

            const results =  await this.sendModifications();

            results.length && this.fetchItems();

            /**
             * Documents can only be deleted from this view
             * They can be edited(editing their data/products) in `DocumentEditOne` view
             */
            const [deleteReq] = results;

            if (deleteReq.message) {
                this.openModalBox(capitalize(deleteReq.message));
            }
 
            if (this.deletedItems.size) {
                this.sendDeletedHistoryData();
                this.refetchMainOverview();
            }

            this.resetCUDItems();
        },

        onCancelChanges () {
            console.log('cancel');

            this.resetCUDItems();
            this.resetChanges();
        },
    },

    computed: {

        chosenProducts () {
            const products = {};

            this.createdItems.forEach(item => {
                if (item.product_name && item.product_name.id) {
                    products[item.product_name.id] = true;
                }
            });

            return products;
        },

        productsAsList () {
            return this.products.filter(({ id }) => !this.chosenProducts[id]);
        },

        ...mapGetters({
            items: 'getItemsAsArr',
            createdItems: 'getCreatedItemsAsArr',
            deletedItems: 'getDeletedItems',
            shouldDisplayConfirmCancelButtons: 'getWhetherItShouldEnableConfirmBtn'
        }),

        ...mapGettersProvider({ providers: 'getItemsAsArr' }),

        ...mapGettersProduct({ products: 'getItemsAsArr', productsMap: 'getItems' }),

        containsErrors () {
            this.errorMessage = this.providers && !this.providers.length 
                ? 'Providers'
                : this.products && !this.products.length 
                    ? 'Products'
                    : this.items && !this.items.length 
                        ? 'Documents'
                        : null

            return this.errorMessage !== null
        },

        vat () {
           return this.$store.getters['dashboard/getCurrentVat']
        },
    },

    beforeRouteLeave (to, from, next) {
        this.$store.commit('SET_PROVIDER', null);
        next();   
    },
    
    async created () {
        const promises = [];
        
        if (this.$store && !this.$store.state[entityName]) {
            this.$store.registerModule(entityName, common);

            promises.push(this.fetchItems());
        }

        if (this.$store && !this.$store.state['product']) {
            this.$store.registerModule('product', common);

            promises.push(
                this.fetchItems(
                    this.mainUrl + 'products',
                    'product'
                )
            );
        }

        if (this.$store && !this.$store.state['provider']) {
            this.$store.registerModule('provider', common);
            
            promises.push(
                this.fetchItems(
                    this.mainUrl + 'providers',
                    'provider'
                )
            );
        }

        this.$store.getters['dashboard/needsInit'] && this.$store.dispatch('dashboard/fetchMainOverview');

        await Promise.all(promises);

        this.initialListItemsLen = this.productsAsList.length;

        this.initCreatedItemsWatcher();

        this.isEverythingLoaded = true;
    },
}
</script>

<style lang="scss" scoped>
    $modal-text-color: darken($color: #394263, $amount: 10%);

    @import '../styles/common.scss';
    @import '../styles/modal.scss';

    .l-flex {
        margin-top: 2rem;
        margin-left: 2rem;
        display: flex;
        justify-content: flex-start;
        align-items: flex-end;

        & .c-select {
            margin-left: 5rem;
            padding: 7px;
            border: 1px solid $main-color;
            outline: none;
            background-color: rgba($color: $main-color, $alpha: .1);
            border-radius: 7px;
        }

        & .c-input {
            border-radius: 7px;
            margin-left: 5rem;
            padding: .3rem;
            border: 1px solid #303753;
        }

        .is-base-icon {
            width: 2rem;
            height: 2rem;
            color: green;
        }
    }
</style>