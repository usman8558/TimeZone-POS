<template>
  <v-row justify="center">
    <v-dialog v-model="closingDialog" max-width="900px" persistent>
      <v-card elevation="8" class="closing-dialog-card">
        <!-- Header -->
        <v-card-title class="closing-header pa-6">
          <div class="header-content">
            <div class="header-icon-wrapper">
              <v-icon :color="isDarkTheme ? 'white' : 'primary'" class="header-icon" size="40">mdi-store-clock-outline</v-icon>
            </div>
            <div class="header-text">
              <h3 :class="['header-title', isDarkTheme ? 'text-white' : '']">{{ __('Closing POS Shift') }}</h3>
              <p :class="['header-subtitle', isDarkTheme ? 'text-grey-lighten-1' : '']">{{ __('Reconcile payment methods and close shift') }}</p>
            </div>
          </div>
        </v-card-title>

        <v-divider class="header-divider"></v-divider>

        <v-card-text :class="['pa-0', isDarkTheme ? 'dark-card-content' : 'white-background']">
          <v-container class="pa-6">
            <!-- ?? Cash Denomination Section -->
            <div class="mb-2 px-2 py-1">
              <h4 :class="['text-h6 mb-1', isDarkTheme ? 'text-white' : 'text-grey-darken-2']">{{ __(' Cash Denominations') }}</h4>
              <v-table density="compact" class="elevation-0 cash-denomination-table">
                <thead>
                  <tr>
                    <th class="bg-color">Currency</th>
                    <th class="bg-color">Denomination</th>
                    <th class="bg-color">Sub Total</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(row, index) in denominationRows" :key="index">
                    <td>{{ row.currency }}</td>
                    <td>
                      <v-text-field
                        v-model.number="row.denomination"
                        type="number"
                        density="compact"
                        hide-details
                        variant="outlined"
                        class="ma-0"
                        :bg-color="isDarkTheme ? '#2A2A2A' : '#EFEFEF'"
                        style="max-width: 100px;"
                        @input="row.sub_total = row.currency * row.denomination"
                      />
                    </td>
                    <td>
                      <v-text-field
                        v-model.number="row.sub_total"
                        type="number"
                        readonly
                        density="compact"
                        hide-details
                        variant="outlined"
                        class="ma-0"
                        :bg-color="isDarkTheme ? '#2A2A2A' : '#EFEFEF'"
                        style="max-width: 100px;"
                      />
                    </td>
                  </tr>
                </tbody>
              </v-table>
            </div>

            <!-- ?? Payment Reconciliation Table -->
            <v-row>
              <v-col cols="12" class="pa-1">
                <div class="table-header mb-4">
                  <h4 :class="['text-h6 mb-1', isDarkTheme ? 'text-white' : 'text-grey-darken-2']">{{ __('Payment Reconciliation') }}</h4>
                  <p class="text-body-2 text-grey">{{ __('Verify closing amounts for each payment method') }}</p>
                </div>

                <v-data-table
                  :headers="headers"
                  :items="dialog_data.payment_reconciliation"
                  item-key="mode_of_payment"
                  :class="['elevation-0 rounded-lg', isDarkTheme ? 'dark-table' : 'white-table']"
                  :items-per-page="itemsPerPage"
                  hide-default-footer
                  density="compact"
                >
                  <template v-slot:item.closing_amount="{ item }">
                    <v-text-field
                      v-model="item.closing_amount"
                      :rules="[max25chars]"
                      :label="frappe._('Edit')"
                      single-line
                      counter
                      type="number"
                      density="compact"
                      variant="outlined"
                      color="primary"
                      bg-color="white"
                      hide-details
                      :prefix="currencySymbol(pos_profile.currency)"
                    />
                  </template>

                  <template v-slot:item.difference="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{ (item.difference = formatCurrency(item.expected_amount - item.closing_amount)) }}
                  </template>

                  <template v-slot:item.opening_amount="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{ formatCurrency(item.opening_amount) }}
                  </template>

                  <template v-slot:item.expected_amount="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{ formatCurrency(item.expected_amount) }}
                  </template>
                </v-data-table>
              </v-col>
            </v-row>
          </v-container>

          <v-container class="pa-6">
          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model.number="totalRevenue"
                :label="__('Total Expense')"
                type="number"
                density="compact"
                variant="outlined"
                :prefix="currencySymbol(pos_profile.currency)"
                hide-details
              />
            </v-col>
          </v-row>
        </v-container>
        </v-card-text>

        <v-divider></v-divider>

        <!-- Footer Buttons -->
        <v-card-actions class="dialog-actions-container">
          <v-btn color="success" variant="flat" class="pos-action-btn" size="large" elevation="2" @click="submit_dialog">
            <v-icon start>mdi-check-circle-outline</v-icon>
            <span>{{ __('Submit') }}</span>
          </v-btn>
          <v-btn color="info" variant="flat" class="pos-action-btn" size="large" elevation="2" @click="print_summary">
            <v-icon start>mdi-printer</v-icon>
            <span>Print</span>
          </v-btn>
          <v-spacer></v-spacer>
          <v-btn theme="dark" @click="close_dialog" class="pos-action-btn cancel-action-btn" size="large" elevation="2">
            <v-icon start>mdi-close-circle-outline</v-icon>
            <span>{{ __('Close') }}</span>
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import format from '../../format';

export default {
  mixins: [format],
  data: () => ({
    closingDialog: false,
    itemsPerPage: 20,
    totalRevenue: 0,
    dialog_data: {},
    pos_profile: '',
    denominationRows: [
      { currency: 5000, denomination: 0, sub_total: 0 },
      { currency: 1000, denomination: 0, sub_total: 0 },
      { currency: 500, denomination: 0, sub_total: 0 },
      { currency: 100, denomination: 0, sub_total: 0 },
      { currency: 50, denomination: 0, sub_total: 0 },
      { currency: 10, denomination: 0, sub_total: 0 },
      { currency: 1, denomination: 0, sub_total: 0 }
    ],
    headers: [
      { title: __('Mode of Payment'), value: 'mode_of_payment', align: 'start' },
      { title: __('Opening Amount'), value: 'opening_amount', align: 'end' },
      { title: __('Closing Amount'), value: 'closing_amount', align: 'end' }
    ],
    max25chars: (v) => String(v).length <= 20 || 'Input too long!',
  }),
  computed: {
      isDarkTheme() {
        return this.$theme.current === 'dark';
      }
    },
  watch: {
    denominationRows: {
      handler() {
        const totalCash = this.denominationRows.reduce(
          (sum, row) => sum + (Number(row.sub_total) || 0),
          0
        );
        const cashRow = this.dialog_data?.payment_reconciliation?.find(
          (item) => item.mode_of_payment?.toLowerCase() === 'cash'
        );
        if (cashRow) cashRow.closing_amount = totalCash;
      },
      deep: true,
      immediate: true,
    },
  },
  methods: {
    close_dialog() {
      this.closingDialog = false;
    },
    submit_dialog() {
      this.dialog_data.totalRevenue = this.totalRevenue;
      this.eventBus.emit('submit_closing_pos', this.dialog_data);
      this.closingDialog = false;
    },
    print_summary() {
  const printWindow = window.open('', '_blank');
  if (!printWindow) {
    frappe.msgprint(__('Please allow pop-ups for this site to print the summary'));
    return;
  }

  const currency = this.pos_profile.currency;
  
  // Find cash payment method
  const cashPayment = this.dialog_data.payment_reconciliation.find(
    item => item.mode_of_payment?.toLowerCase() === 'cash'
  );

  
  const bankPayment = this.dialog_data.payment_reconciliation.find(
  item => item.mode_of_payment?.toLowerCase() === 'bank'
);


  
  // Get the expected amount for cash
  const cashExpectedAmount = cashPayment?.expected_amount || 0;
  const bankClosingAmount = bankPayment?.closing_amount || 0;

  
  // Calculate actual cash from denominations
  const cashActual = this.denominationRows.reduce(
    (sum, row) => sum + (Number(row.sub_total) || 0), 0
  );
  

  

  // Calculate total sales
  const totalSales = this.dialog_data.payment_reconciliation.reduce(
    (sum, row) => sum + (parseFloat(row.closing_amount) || 0), 0
  );

  

  // Sirf bank ki closing amount ka total
const bankClosingTotal = this.dialog_data.payment_reconciliation
  .filter(row => row.mode_of_payment?.toLowerCase() === 'bank')
  .reduce((sum, row) => sum + (parseFloat(row.closing_amount) || 0), 0);


  //Calculate expected amount from Closing Box (All payment methods)
  //Calculate expected amount from Closing Box (All payment methods)
  // Ensure that no negative values or miscalculated expected amounts affect the total
  const totalExpected = this.dialog_data.payment_reconciliation.reduce(
    (sum, row) => {
        let expected = parseFloat(row.expected_amount) || 0;
        return sum + expected;
    }, 0
  );

  

  // Build payment rows
  const paymentRows = this.dialog_data.payment_reconciliation
    .map(item => `
      <tr>
        <td>${item.mode_of_payment}</td>
        <td>${currency} ${item.expected_amount || '0.00'}</td>
      </tr>
    `).join('');


  // Build denomination rows
   const denominationRows = this.denominationRows
    .map((row, index) => `
      <tr style="background-color: ${index % 2 === 0 ? '#ffffff' : '#f8f8f8'}">
        <td>${row.currency}</td>
        <td>${row.denomination}</td>
        <td>${row.currency * row.denomination}</td>
      </tr>
    `).join('');

  const currentUser = frappe.session.user || "N/A";
  const userEmail = frappe.session.user_email || "abc@gmail.com";
  const location = this.pos_profile.name || "N/A";
  const totalRevenue = this.totalRevenue || 0;
  const netSales = totalExpected - totalRevenue

  const difference = netSales - totalSales;
  const currentTime = frappe.datetime.now_datetime();

  printWindow.document.write(`
    <html>
      <head>
        <title>POS Closing Summary</title>
        <style>
          body { font-family: Arial, sans-serif; padding: 20px; }
          h2, h3 { margin-bottom: 5px; }
          table { width: 100%; border-collapse: collapse; margin-top: 10px; }
          th, td { border: 1px solid #ccc; padding: 8px; font-size: 14px; }
          th { background-color: #efefef; text-align: left; }
          .section-title { margin-top: 30px; font-weight: bold; }
          .footer { margin-top: 30px; font-size: 14px; }
          @media print {
          body { -webkit-print-color-adjust: exact !important; }
        }
        .powered-by { 
          margin-top: 40px;
          text-align: center;
          font-size: 12px;
          color: #666;
          border-top: 1px solid #eee;
          padding-top: 10px;
          display: flex;
          align-items: center;
          justify-content: center;
          gap: 8px;
        }
        .powered-by img {
          height: 20px;
          width: auto;
        }
        </style>
      </head>
      <body>
        <h2>POS - ${location}</h2>
        <p>Current Register (${currentTime})</p>

        <h3 class="section-title">Payment Method</h3>
        <table>
          <thead>
            <tr style="background:#f4f4f4">
              <th>Payment Method</th>
              <th>Sell</th>
              
            </tr>
          </thead>
          <tbody>
            ${paymentRows}
          </tbody>
        </table>

        <h3 class="section-title">Summary</h3>
        <table>
          <tbody>
            <tr><td>Gross Sales</td><td>${currency} ${totalExpected.toFixed(2)}</td></tr>
            <tr><td>Total Refund</td><td>${currency} 0.00</td></tr>
            <tr><td>Total Payment</td><td>${currency} ${totalExpected.toFixed(2)}</td></tr>
            <tr><td>Credit Sales</td><td>${currency} 0.00</td></tr>
            <tr><td>Total Expense</td><td>${currency} ${totalRevenue.toFixed(2)}</td></tr>
            <tr><td>Net Sales</td><td>${currency} ${this.formatCurrency(netSales)}</td></tr>
            
          </tbody>
        </table>

        <h3 class="section-title">Cash Denominations</h3>
        <table>
          <thead>
            <tr style="background:#f4f4f4">
              <th>Denomination</th>
              <th>Count</th>
              <th>Subtotal</th>
            </tr>
          </thead>
          <tbody>
            ${denominationRows}
            <tr style="background:#f4f4f4"><td colspan="2"><strong>Total</strong></td><td><strong>${currency} ${cashActual.toFixed(2)}</strong></td></tr>
            <tr style="background:#f4f4f4">
                  <td colspan="2"><strong>Card Sales</strong></td>
                  <td><strong>${currency} ${bankClosingTotal.toFixed(2)}</strong></td>
                </tr>

          </tbody>
        </table>

        <h3 class="section-title">Closing Details</h3>
        <table>
          <tbody>
            <tr><td>Expected Closing</td><td>${currency} ${netSales.toFixed(2)}</td></tr>
            <tr><td>Actual Closing </td><td>${currency} ${totalSales.toFixed(2)}</td></tr>
            <tr><td>Difference</td><td>${currency} ${difference.toFixed(2)}</td></tr>
          </tbody>
        </table>

        <div class="footer">
          <p><strong>User:</strong> ${currentUser}</p>
          <p><strong>Email:</strong> ${userEmail}</p>
          <p><strong>Business Location:</strong> ${location}</p>
        </div>
         <div class="powered-by">
        <span>Powered by</span>
        <img src="https://erp.truetechterminal.com/files/Tech-Accuracy-04-1.png" alt="TechAccuracy Logo">
      </div>
      </body>
    </html>
  `);

  printWindow.document.close();
  setTimeout(() => {
    printWindow.focus();
    printWindow.print();
  }, 500);
}
  },
  created() {
    this.eventBus.on('open_ClosingDialog', (data) => {
      this.closingDialog = true;
      this.dialog_data = data;
    });
    this.eventBus.on('register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
      const restricted_users = ['kings1@gmail.com','frozen1@gmail.com','nbmainbranch1@gmail.com','danyouchouk1@gmail.com','nbkhomar1@gmail.com','kashrote1@gmail.com'];
      const current_user = frappe.session.user;
      const should_hide = restricted_users.includes(current_user);
      this.pos_profile.hide_expected_amount = should_hide;

      if (!should_hide) {
        this.headers.push(
          { title: __('Expected Amount'), value: 'expected_amount', align: 'end' },
          { title: __('Difference'), value: 'difference', align: 'end' }
        );
      }
    });
  },
  beforeUnmount() {
    this.eventBus.off('open_ClosingDialog');
    this.eventBus.off('register_pos_profile');
  }
};
</script>

<style scoped>
/* Dark Mode Specific Styles */
.dark-card-content {
  background-color: #1E1E1E !important;
  color: #FFFFFF !important;
}

.dark-table {
  background-color: transparent !important;
  color: white !important;
}

:deep(.v-theme--dark) .v-table {
  background-color: transparent !important;
  color: white !important;
}

:deep(.v-theme--dark) .v-table th.bg-color {
  background-color: #2A2A2A !important;
  color: #BB86FC !important; /* Header text color */
}

.v-text-field :deep(input) {
  color: inherit !important;
}
.v-table td,
.v-table th {
  padding: 3px !important;
}
.v-table input {
  font-size: 12px;
  padding: 3px !important;
}
.bg-color {
  background-color: #EFEFEF !important;
}
.cash-denomination-table th,
.cash-denomination-table td {
  font-size: 13px;
}
</style>
