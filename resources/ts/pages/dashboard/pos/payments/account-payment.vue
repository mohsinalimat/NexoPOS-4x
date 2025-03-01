<template>
    <div class="h-full w-full py-2">
        <div class="px-2 pb-2" v-if="order">
            <div class="grid grid-cols-2 gap-2">
                <div id="details" class="h-16 flex justify-between items-center bg-blue-400 text-white text-xl md:text-3xl p-2">
                    <span>{{ __( 'Total' ) }} : </span>
                    <span>{{ order.total | currency }}</span>
                </div>
                <div id="discount" @click="toggleDiscount()" class="cursor-pointer h-16 flex justify-between items-center bg-red-400 text-white text-xl md:text-3xl p-2">
                    <span>{{ __( 'Discount' ) }} : </span>
                    <span>{{ order.discount | currency }}</span>
                </div>
                <div id="paid" class="h-16 flex justify-between items-center bg-green-400 text-white text-xl md:text-3xl p-2">
                    <span>{{ __( 'Paid' ) }} : </span>
                    <span>{{ order.tendered | currency }}</span>
                </div>
                <div id="change" class="h-16 flex justify-between items-center bg-teal-400 text-white text-xl md:text-3xl p-2">
                    <span>{{ __( 'Change' ) }} : </span>
                    <span>{{ order.change | currency }}</span>
                </div>
                <div id="change" class="col-span-2 h-16 flex justify-between items-center bg-blue-400 text-white text-xl md:text-3xl p-2">
                    <span>{{ __( 'Current Balance' ) }} : </span>
                    <span>{{ order.customer.account_amount | currency }}</span>
                </div>
                <div id="change" class="col-span-2 h-16 flex justify-between items-center bg-gray-300 text-gray-800 text-xl md:text-3xl p-2">
                    <span>{{ __( 'Screen' ) }} : </span>
                    <span>{{ screenValue | currency }}</span>
                </div>
            </div>
        </div>
        <div class="px-2 pb-2">
            <div class="-mx-2 flex flex-wrap">
                <div class="pl-2 pr-1 flex-auto">
                    <ns-numpad :floating="true" @changed="handleChange( $event )" @next="proceedAddingPayment( $event )">
                        <template v-slot:numpad-footer>
                            <div
                            @click="makeFullPayment()"
                            class="hover:bg-green-500 col-span-3 bg-green-400 text-2xl text-white border h-16 flex items-center justify-center cursor-pointer">
                            {{ __( 'Full Payment' ) }}</div>
                        </template>
                    </ns-numpad>
                </div>
                <div class="w-1/2 md:w-72 pr-2 pl-1">
                    <div class="grid grid-flow-row grid-rows-1 gap-2">
                        <div 
                            @click="increaseBy({ value : 100 })"
                            class="hover:bg-gray-400 hover:text-gray-800 bg-gray-300 text-2xl text-gray-700 border h-16 flex items-center justify-center cursor-pointer">
                            <span>{{ 100 | currency }}</span>
                        </div>
                        <div 
                            @click="increaseBy({ value : 500 })"
                            class="hover:bg-gray-400 hover:text-gray-800 bg-gray-300 text-2xl text-gray-700 border h-16 flex items-center justify-center cursor-pointer">
                            <span >{{ 500 | currency }}</span>
                        </div>
                        <div 
                            @click="increaseBy({ value : 1000 })"
                            class="hover:bg-gray-400 hover:text-gray-800 bg-gray-300 text-2xl text-gray-700 border h-16 flex items-center justify-center cursor-pointer">
                            <span >{{ 1000 | currency }}</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script>
import { default as nsNumpad } from "@/components/ns-numpad";
import { nsSnackBar } from '@/bootstrap';
import nsPosConfirmPopupVue from '@/popups/ns-pos-confirm-popup.vue';
import { __ } from '@/libraries/lang';

export default {
    name: "ns-account-payment",
    components: {
        nsNumpad
    },
    props: [ 'identifier', 'label' ],
    data() {
        return {
            subscription: null,
            screenValue: 0,
            order: null,
        }
    },
    methods: {
        __,

        handleChange( event ) {
            this.screenValue    =   event;
        },
        proceedAddingPayment( event ) {
            const value    =   parseFloat( event );
            const payments  =   this.order.payments;

            if ( value <= 0 ) {
                return nsSnackBar.error( __( 'Please provide a valid payment amount.' ) )
                    .subscribe();
            }

            if ( payments.filter( p => p.identifier === 'account-payment' ).length > 0 ) {
                return nsSnackBar.error( __( 'The customer account can only be used once per order. Consider deleting the previously used payment.' ) )
                    .subscribe();
            }

            if ( value > this.order.customer.account_amount ) {
                return nsSnackBar.error( __( 'Not enough funds to add {amount} as a payment. Available balance {balance}.' )
                    .replace( '{amount}', this.$options.filters.currency( value ) ) 
                    .replace( '{balance}', this.$options.filters.currency( this.order.customer.account_amount ) ) 
                ).subscribe();
            }

            POS.addPayment({
                value,
                identifier: 'account-payment',
                selected: false,
                label: this.label,
                readonly: false,
            });

            this.order.customer.account_amount  -=  value;
            POS.selectCustomer( this.order.customer );

            this.$emit( 'submit' );
        },
        proceedFullPayment() {
            this.proceedAddingPayment( this.order.total );
        },
        makeFullPayment() {
            Popup.show( nsPosConfirmPopupVue, {
                title: __( 'Confirm Full Payment' ),
                message: __( 'You\'re about to use {amount} from the customer account to make a payment. Would you like to proceed ?' ).replace( '{amount}', this.$options.filters.currency( this.order.total ) ),
                onAction: ( action ) => {
                    if ( action ) {
                        this.proceedFullPayment();
                    }
                }
            });
        },
    },
    mounted() {
        this.subscription   =   POS.order.subscribe( order => this.order = order );
    },
    destroyed() {
        this.subscription.unsubscribe();
    }
}
</script>