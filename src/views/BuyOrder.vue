<template>
  <main>
    <div class="secondary">
      <v-card-title>
        {{ business_type == 'buy' ? 'ĐƠN MUA' : 'ĐƠN BÁN'}} {{ formatDate2(date) }}
        <v-btn outlined class="ml-5" color="primary" dense>Tổng Usdt: {{ formatUSDT(total_usdt) }}</v-btn>
        <v-btn outlined class="ml-5" color="orange" dense>Tổng VND: {{ formatMoney(total_money) }}</v-btn>
        <v-spacer></v-spacer>
        <div class="w300">
          <v-select :items="types" v-model="business_type" item-text="title" item-value="value" label="Giao dịch" outlined
            dense></v-select>
        </div>
        <v-btn class="ml-5 mb-1" @click="exportReport" :disabled="loading">Xuất Excel</v-btn>
        <div class="w300 ml-5">
          <v-menu v-model="menu2" :close-on-content-click="false" transition="scale-transition" offset-y min-width="auto">
            <template v-slot:activator="{ on, attrs }">
              <v-text-field v-model="date" label="Chọn ngày" append-icon="mdi-calendar" readonly v-bind="attrs" v-on="on"
                outlined dense></v-text-field>
            </template>
            <v-date-picker v-model="date" @input="menu2 = false" locale="vi-vn"></v-date-picker>
          </v-menu>
        </div>
        <div class="w300 ml-5">
          <v-text-field v-model="search" append-icon="mdi-magnify" label="Tìm mã đơn" @keydown.enter="searchHandle"
            outlined dense></v-text-field>
        </div>
        <v-btn class="mb-1 ml-5" icon outlined @click="logout"><v-icon>mdi-arrow-right</v-icon></v-btn>
      </v-card-title>
      <v-data-table :headers="headers" :items="data" :items-per-page="1000"
        :footer-props="{ 'items-per-page-options': [100, 1000] }" :hide-default-footer="true">
        <template v-slot:[`item.total_money`]="{ item }">
          {{ formatMoney(item.total_money) }} ₫
        </template>
        <template v-slot:[`item.amount`]="{ item }">
          {{ formatUSDT(item.amount) }}
        </template>
        <template v-slot:[`item.rate`]="{ item }">
          {{ formatMoney(item.rate) }}
        </template>
        <template v-slot:[`item.paid_money`]="{ item }">
          <div v-if="item.paid_money">
            <div v-for="(i, index) in parseArr(item.paid_money)" :key="index">{{ formatMoney(i) }} ₫</div>
          </div>
        </template>
        <template v-slot:[`item.bank_name`]="{ item }">
          <div v-if="item.bank_name">
            <div v-for="(i, index) in parseArr(item.bank_name)" :key="index">{{ i }}</div>
          </div>
        </template>
        <template v-slot:[`item.debt`]="{ item }">
          {{ formatMoney(item.debt) }} ₫
        </template>
        <template v-slot:[`item.created_at`]="{ item }">
          {{ formatDate(item.created_at) }}
        </template>
        <template v-slot:[`item.actions`]="{ item }">
          <v-btn small class="error mr-2" @click="removeOrder(item.code)">
            Xóa
          </v-btn>
        </template>
      </v-data-table>
    </div>
    <v-dialog v-model="dialogDelete" max-width="500px">
      <v-card>
        <v-card-title class="headline">
          Xác nhận xóa
        </v-card-title>
        <v-card-actions>
          <v-spacer />
          <v-btn color="blue darken-1" text @click="dialogDelete = false">
            Hủy
          </v-btn>
          <v-btn color="blue darken-1" text @click="removeConfirm">
            Xác nhận
          </v-btn>
          <v-spacer />
        </v-card-actions>
      </v-card>
    </v-dialog>
  </main>
</template>

<script>
import { mapGetters } from "vuex";
export default {
  data() {
    return {
      search: "",
      menu2: false,
      dialogDelete: false,
      headers: [
        { text: "Thời gian", value: "created_at" },
        { text: "NVKD", value: "seller" },
        { text: "Khách hàng", value: "client" },
        { text: "Số lượng", value: "amount" },
        { text: "Rate", value: "rate" },
        { text: "Thành tiền", value: "total_money" },
        { text: "Đã chuyển/nhận", value: "paid_money" },
        { text: "Công nợ", value: "debt" },
        { text: "Tên Bank", value: "bank_name" },
        { text: "Thao tác", value: "actions", sortable: false },
      ],
      data: [],
      edit_id: "",
      excel_htmls: '',
      date: (new Date(Date.now() - (new Date()).getTimezoneOffset() * 60000)).toISOString().substr(0, 10),
      loading: false,
      total_money: 0,
      total_usdt: 0,
      business_type: '',
      types: [
        {
          title: "Đơn mua",
          value: 'buy'
        },
        {
          title: "Đơn bán",
          value: 'sell'
        },
      ]
    };
  },
  computed: {
    ...mapGetters(["account"]),
  },
  mounted() {
    this.business_type = this.$route.params.id
  },
  methods: {
    getData() {
      this.loading = true
      let url = `sales?type=${this.business_type}&page=1&perPage=999&from=${this.date}&to=${this.date}`
      this.CallAPI("get", url, {}, (res) => {
        this.data = res.data.data;
        this.getDataExport(this.data)
        this.loading = false
      });
    },
    removeConfirm() {
      this.CallAPI("delete", "sales/" + this.edit_id, {}, (res) => {
        this.$toast.success('Xóa thành công')
        this.getData()
        this.dialogDelete = false
      });
    },
    removeOrder(code) {
      this.edit_id = code
      this.dialogDelete = true
    },
    searchHandle() {
      this.CallAPI("get", "search/sales?query=" + this.search, {}, (res) => {
        this.data = res.data;
      });
    },
    parseArr(arr) {
      let array = []
      for (let i in JSON.parse(arr)) {
        array.push(JSON.parse(arr)[i])
      }
      return array
    },
    formatDate(date) {
      let d = new Date(date);
      return d.toLocaleString();
    },
    formatDate2(date) {
      return `${date.split("-")[2]}/${date.split("-")[1]}/${date.split("-")[0]}`
    },
    formatMoney(value) {
      if (!value) return 0;
      return String(parseFloat(value).toFixed(0))
        .toString()
        .replace(/\B(?=(\d{3})+(?!\d))/g, ",");
    },
    formatUSDT(value) {
      if (!value) return 0;
      return String(parseFloat(value).toFixed(1))
        .toString()
        .replace(/\B(?=(\d{3})+(?!\d))/g, ",");
    },
    totalAmount(data, token) {
      let arr = data.filter((item) => item.token == token)
      let total = 0
      let rate_total = 0
      arr.forEach(item => {
        total += item.amount
        rate_total += item.rate * item.amount
      });
      return {
        total: total,
        average: rate_total / total
      }
    },
    getDataExport(data) {
      this.total_money = 0
      this.total_usdt = 0
      this.excel_htmls = `
            <tr>
              <td colspan="9" style="text-align: center"><b>BÁO CÁO ĐƠN ${ this.business_type == 'buy' ? 'MUA' : 'BÁN'} ${this.date}</b></td>
            </tr>
            <tr>
                <th style="width: 150px">THỜI GIAN</th>
                <th style="width: 150px">NVKD</th>
                <th style="width: 200px">KHÁCH HÀNG</th>
                <th style="width: 100px">SỐ LƯỢNG</th>
                <th style="width: 80px">RATE</th>
                <th style="width: 150px">THÀNH TIỀN</th>
                <th style="width: 150px">ĐÃ CHUYỂN/NHẬN</th>
                <th style="width: 150px">CÔNG NỢ</th>
                <th style="width: 200px">TÊN BANK</th>
            </tr>
        `;

      for (let item of data) {
        let paid_money = this.parseArr(item.paid_money)
        let bank_name = this.parseArr(item.bank_name)
        let len = paid_money.length
        this.excel_htmls += `
                <tr>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${this.formatDate(item.created_at)}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${item.seller}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${item.client}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${this.formatUSDT(item.amount)}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${this.formatMoney(item.rate)}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${this.formatMoney(item.total_money)}</td>
                    <td>${this.formatMoney(paid_money[0])}</td>
                    <td rowspan="${len}" style="text-align: center; vertical-align: middle;">${this.formatMoney(item.debt)}</td>
                    <td>${bank_name[0] || ''}</td>
                </tr>
            `;

        for (let i = 1; i < len; i++) {
          this.excel_htmls += `
                <tr>
                    <td>${this.formatMoney(paid_money[i])}</td>
                    <td>${bank_name[i]}</td>
                </tr>
            `;
        }

        this.total_money += item.total_money
        this.total_usdt += item.amount
      }

      this.excel_htmls += `
            <tr>
              <td colspan="3" style="text-align: center; vertical-align: middle; height: 50px; background: #ffea52"><b>Tổng/Trung bình Rate</b></td>
              <td style="text-align: center; vertical-align: middle; background: #ffea52"><b>${this.formatUSDT(this.total_usdt)}</b></td>
              <td style="text-align: center; vertical-align: middle; background: #ffea52"><b>${this.formatMoney(this.total_money / this.total_usdt)}</b></td>
              <td style="text-align: center; vertical-align: middle; background: #ffea52"><b>${this.formatMoney(this.total_money)}</b></td>
              <td colspan="3" style="background: #ffea52"></td>
            </tr>
        `
    },
    exportReport() {
      var uri = "data:application/vnd.ms-excel;base64,";
      var template =
        '<html xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:x="urn:schemas-microsoft-com:office:excel" xmlns="http://www.w3.org/TR/REC-html40"><head><meta charset="UTF-8"><!--[if gte mso 9]><xml><x:ExcelWorkbook><x:ExcelWorksheets><x:ExcelWorksheet><x:Name>{worksheet}</x:Name><x:WorksheetOptions><x:DisplayGridlines/></x:WorksheetOptions></x:ExcelWorksheet></x:ExcelWorksheets></x:ExcelWorkbook></xml><![endif]--></head><body><table border>{table}</table></body></html>';
      var base64 = function (s) {
        return window.btoa(unescape(encodeURIComponent(s)));
      };

      var format = function (s, c) {
        return s.replace(/{(\w+)}/g, function (m, p) {
          return c[p];
        });
      };

      var ctx = {
        worksheet: "Worksheet",
        table: this.excel_htmls,
      };

      var link = document.createElement("a");
      link.download = `Báo cáo đơn ${ this.business_type == 'buy' ? 'mua' : 'bán'} ngày ${this.date}.xls`
      link.href = uri + base64(format(template, ctx));
      link.click();

      // const data = new FormData();
      // data.append("file", this.dataURLtoFile(link.href, 'export.xls'));
      // this.CallAPI("post", "manage/upload-file", data, (res) => {
      //   window.prompt("File Link", this.image(res.data))
      // })
    },
    dataURLtoFile(dataurl, filename) {
      var arr = dataurl.split(','),
        mime = arr[0].match(/:(.*?);/)[1],
        bstr = atob(arr[arr.length - 1]),
        n = bstr.length,
        u8arr = new Uint8Array(n);
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      return new File([u8arr], filename, { type: mime });
    },
    logout() {
      localStorage.removeItem("access_token");
      this.$router.push("/login");
    },
  },
  watch: {
    search() {
      if (!this.search) {
        this.getData()
      }
    },
    date() {
      this.getData()
    },
    business_type() {
      this.$router.push(this.business_type)
      this.getData()
    },
    $route() {
      this.business_type = this.$route.params.id
    }
  },
};
</script>