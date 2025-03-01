<template>
    <div class="shadow-lg h-95vh md:h-5/6-screen lg:h-5/6-screen w-95vw md:w-4/6-screen lg:w-3/6-screen bg-white flex flex-col">
        <div class="p-2 border-b flex justify-between items-center">
            <h3 class="font-semibold">{{ __( 'Layaway Parameters' ) }}</h3>
            <div>
                <ns-close-button @click="close()"></ns-close-button>
            </div>
        </div>
        <div class="p-2 flex-auto flex flex-col relative overflow-y-auto">
            <div v-if="fields.length === 0" class="absolute h-full w-full flex items-center justify-center">
                <ns-spinner></ns-spinner>
            </div>
            <div class="p-2 text-white bg-blue-400 mb-2 text-center text-2xl font-bold flex justify-between">
                <span>{{ __( 'Minimum Payment' ) }}</span>
                <span>{{ expectedPayment | currency }}</span>
            </div>
            <div>
                <ns-field v-for="( field, index ) of fields" :field="field" :key="index"></ns-field>
            </div>
            <div class="flex flex-col flex-auto overflow-hidden">
                <div class="border-b border-gray-200">
                    <h3 class="text-2xl flex justify-between py-2 text-gray-700">
                        <span>{{ __( 'Instalments & Payments' ) }}</span>
                        <p>
                            <span class="text-sm">({{ totalPayments | currency }})</span>
                            <span>
                            {{ order.total | currency }}
                            </span>
                        </p>
                    </h3>
                    <p class="p-2 mb-2 text-center bg-green-200 text-green-700">
                        {{ __( 'The final payment date must be the last within the instalments.' ) }}
                    </p>
                </div>
                <div class="flex-auto overflow-y-auto">
                    <div class="flex w-full -mx-1 py-2" :key="key" v-for="(instalment, key) of order.instalments">
                        <div class="flex flex-auto">
                            <div class="px-1 w-full md:w-1/2">
                                <ns-field @change="refreshTotalPayments()" :field="instalment.date"></ns-field>
                            </div>
                            <div class="px-1 w-full md:w-1/2">
                                <ns-field @change="refreshTotalPayments()" :field="instalment.amount"></ns-field>
                            </div>
                        </div>
                        <div class="flex items-center">
                            <button class="items-center flex justify-center h-8 w-8 rounded border border-gray-200 hover:bg-red-400 hover:border-red-400 hover:text-white">
                                <i class="las la-times"></i>
                            </button>
                        </div>
                    </div>
                    <div class="my-2" v-if="order.instalments.length === 0">
                        <p class="p-2 bg-gray-200 text-gray-700 text-center">{{ __( 'There is not instalment defined. Please set how many instalments are allowed for this order' ) }}</p>
                    </div>
                </div>
            </div>
        </div>
        <div class="p-2 flex border-t justify-between flex-shrink-0">
            <div></div>
            <div class="-mx-1 flex">
                <div class="px-1">
                    <ns-button @click="close()" type="danger">{{ __( 'Cancel' ) }}</ns-button>
                </div>
                <div class="px-1">
                    <ns-button @click="updateOrder()" type="info">{{ __( 'Proceed' ) }}</ns-button>
                </div>
            </div>
        </div>
    </div>
</template>
<script>
import FormValidation from '@/libraries/form-validation';
import { nsHttpClient, nsSnackBar } from '@/bootstrap';
import { __ } from '@/libraries/lang';
export default {
    name: 'ns-pos-layaway-popup',
    data() {
        return {
            fields: [],
            instalments: [],
            formValidation: new FormValidation,
            subscription: null,
            totalPayments: 0
        }
    },
    mounted() {
        this.loadFields();
        this.subscription   =   this.$popup.event.subscribe( action => {
            if ([ 'click-overlay', 'press-esc' ].includes( action.event ) ) {
                this.close();
            }
        });        
    },
    updated() {
        setTimeout( () => {
            document.querySelector( '.is-popup #total_instalments' ).addEventListener( 'change', () => {
                const totalInstalments    =   this.formValidation.extractFields( this.fields ).total_instalments;
                this.generatePaymentFields( totalInstalments );
            });
            document.querySelector( '.is-popup #total_instalments' ).addEventListener( 'focus', () => {
                document.querySelector( '.is-popup #total_instalments' ).select();
            });
        }, 200 );
    },
    computed: {
        expectedPayment() {
            const minimalPaymentPercent     =   this.order.customer.group.minimal_credit_payment;
            return nsRawCurrency( ( this.order.total * minimalPaymentPercent ) / 100 );
        },
        order() {
            this.$popupParams.order.instalments     =   this.$popupParams.order.instalments.map( instalment => {
                for( let name in instalment ) {
                    /**
                     * to avoid performing
                     * this operation multiple time
                     */
                    if ( typeof instalment[ name ] !== 'object' ) {
                        if ( name === 'date' ) {
                            const field    =   {
                                type: 'date',
                                name,
                                label: __( 'Date' ),
                                disabled: instalment.paid === 1 ? true : false,
                                value: moment( instalment.date ).format( 'YYYY-MM-DD' )
                            };
    
                            instalment[ name ]    =   field;
                        } else if ( name === 'amount' ) {
                            const field    =   {
                                type: 'number',
                                name,
                                label: __( 'Amount' ),
                                disabled: instalment.paid === 1 ? true : false,
                                value: instalment.amount
                            };
    
                            instalment[ name ]    =   field;
                        } else if ( ! [ 'paid', 'id' ].includes( name ) ) {
                            const field    =   {
                                type: 'hidden',
                                name,
                                value: instalment[ name ]
                            };
    
                            instalment[ name ]    =   field;
                        }
                    }
                }

                return instalment;
            });

            return this.$popupParams.order;
        },
    },
    destroyed() {
        this.subscription.unsubscribe();
    },
    methods: {
        __,
        refreshTotalPayments() {
            if ( this.order.instalments.length > 0 ) {
                const totalInstalments      =   nsRawCurrency( this.order.instalments
                    .map( i => parseFloat( i.amount.value ) || 0 )
                    .reduce( ( before, after ) => {
                        return parseFloat( before ) + parseFloat( after );
                    }) );
                this.totalPayments          =    this.order.total - totalInstalments;
            } else {
                this.totalPayments  =   0;
            }
        },
        generatePaymentFields( totalInstalments ) {
            this.order.instalments    =   ( new Array( parseInt( totalInstalments ) ) )
                .fill('')
                .map( ( _, index ) => {
                    return {
                        date: {
                            type: 'date',
                            name: 'date',
                            label: 'Date',
                            value: index === 0 ? ns.date.moment.format( 'YYYY-MM-DD' ) : '',
                        },
                        amount: {
                            type: 'number',
                            name: 'amount',
                            label: 'Amount',
                            value: index === 0 ? this.expectedPayment : 0,
                        },
                        readonly : {
                            type: 'hidden',
                            name: 'readonly',
                            value: this.expectedPayment > 0 && index === 0 ? true: false
                        },
                    }
                });

            this.$forceUpdate();
            this.refreshTotalPayments();
        },
        close() {
            this.$popupParams.reject({ status: 'failed', message: __( 'You must define layaway settings before proceeding.' ) });
            this.$popup.close();
        },
        updateOrder() {
            if ( this.order.instalments.length === 0 ) {
                return nsSnackBar.error( __( 'Please provide instalments before proceeding.' ) ).subscribe();
            }
            
            this.fields.forEach( field => this.formValidation.validateField( field ) );

            if ( ! this.formValidation.fieldsValid( this.fields ) ) {
                return nsSnackBar.error( __( 'Unable to procee the form is not valid' ) ).subscribe();
            }

            this.$forceUpdate();

            const instalments           =   this.order.instalments.map( instalment => {
                return {
                    amount  : parseFloat( instalment.amount.value ),
                    date    : instalment.date.value,
                }
            });

            const totalInstalments      =   nsRawCurrency( instalments
                .map( p => p.amount )
                .reduce( (before, after) => parseFloat( before ) + parseFloat( after ) ) );

            if ( instalments.filter( instalment => instalment.date === undefined || instalment.date === '' ).length > 0 ) {
                return nsSnackBar.error( __( 'One or more instalments has an invalid date.' ) ).subscribe();
            }

            if ( instalments.filter( instalment => ! ( instalment.amount > 0 ) ).length > 0 ) {
                return nsSnackBar.error( __( 'One or more instalments has an invalid amount.' ) ).subscribe();
            }

            if ( instalments.filter( instalment => moment( instalment.date ).isBefore( ns.date.moment.startOf( 'day' ) ) ).length > 0 ) {
                return nsSnackBar.error( __( 'One or more instalments has a date prior to the current date.' ) ).subscribe();
            }

            const instalmentsForToday   =   instalments.filter( instalment => moment( instalment.date ).isSame( ns.date.moment.startOf( 'day' ), 'day' ) );
            let totalPaidToday          =   0;


            instalmentsForToday.forEach( instalment => {
                totalPaidToday      +=  parseFloat( instalment.amount );
            });

            if ( totalPaidToday < this.expectedPayment ) {
                return nsSnackBar.error( __( 'The payment to be made today is less than what is expected.' ) ).subscribe();
            }

            if ( totalInstalments < nsRawCurrency( this.order.total ) ) {
                return nsSnackBar.error( __( 'Total instalments must be equal to the order total.' ) ).subscribe();
            }

            instalments.sort( ( before, after ) => {
                const beforeMoment  =   moment( before.date );
                const afterMoment   =   moment( after.date );

                if ( beforeMoment.isBefore( afterMoment ) ) {
                    return -1;
                } else if ( beforeMoment.isAfter( afterMoment ) ) {
                    return 1;
                }

                return 0;
            });

            const fields                =   this.formValidation.extractFields( this.fields );

            fields.final_payment_date   =   instalments.reverse()[0].date;
            fields.total_instalments    =   instalments.length;

            const order                 =   { ...this.$popupParams.order, ...fields, instalments };
            const { resolve, reject }   =   this.$popupParams;

            this.$popup.close();
            
            POS.order.next( order );
            
            return resolve( order );
        },
        loadFields() {
            nsHttpClient.get( `/api/nexopos/v4/fields/ns.layaway` )
                .subscribe( fields => {
                    this.fields     =   this.formValidation.createFields( fields );
                    this.fields.forEach( field => {
                        if ( field.name === 'total_instalments' ) {
                            field.value     =   this.order.total_instalments || 0;
                        }
                    });
                })
        }
    }
}
</script>