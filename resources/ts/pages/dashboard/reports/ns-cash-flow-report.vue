<script>
import moment from "moment";
import nsDatepicker from "@/components/ns-datepicker";
import { default as nsDateTimePicker } from '@/components/ns-date-time-picker';
import { nsHttpClient, nsSnackBar } from '@/bootstrap';


export default {
    name : 'ns-cash-flow',
    mounted() {
    },
    components: {
        nsDatepicker,
        nsDateTimePicker,
    },
    data() {
        return {
            startDate: moment().format( 'YYYY/MM/DD HH:mm' ),
            endDate: moment().format( 'YYYY/MM/DD HH:mm' ),
            report: []
        }
    },
    computed: {
        totalDebit() {
            return 0;
        },
        totalCredit() {
            return 0;
        }
    },
    methods: {
        setStartDate( moment ) {
            this.startDate  =   moment.format( 'YYYY/MM/DD HH:mm' );
        },
        setEndDate( moment ) {
            this.endDate    =   moment.format( 'YYYY/MM/DD HH:mm' );
        },
        loadReport() {
            const startDate     =   this.startDate;
            const endDate       =   this.endDate;

            nsHttpClient.post( '/api/nexopos/v4/reports/cash-flow', { startDate, endDate })
                .subscribe({
                    next: result => {
                        this.report     =   result;
                        console.log( this.report );
                    },
                    error: ( error ) => {
                        nsSnackBar
                            .error( error.message )
                            .subscribe();
                    }
                })
        }
    }
}
</script>