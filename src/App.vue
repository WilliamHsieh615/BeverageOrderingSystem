<template>
  <div id="app" class="container-fluid py-5">
    <div class="row g-4">
      <!-- ================= 產品清單 ================= -->
      <div class="col-md-4">
        <div class="list-group">
          <a
            href="#"
            class="list-group-item list-group-item-action"
            :class="{ active: tempSelected.name === product.name }"
            v-for="product in products"
            :key="product.name"
            @click.prevent="selectedProduct(product)"
          >
            <h6 class="mb-1">{{ product.name }}</h6>
            <div class="d-flex justify-content-between">
              <small>{{ product.engName }}</small>
              <small>NT$ {{ product.price }}</small>
            </div>
          </a>
        </div>
      </div>

      <!-- ================= 產品設定  ================= -->
      <div class="col-md-8">
        <div class="card mb-3">
          <!-- 遮罩：尚未選飲品 -->
          <div
            v-if="!tempSelected.name"
            class="position-absolute top-0 bottom-0 start-0 end-0 d-flex align-items-center justify-content-center text-white fw-bold"
            style="background: rgba(0, 0, 0, 0.55); z-index: 10"
          >
            請先選擇飲品
          </div>

          <div class="card-body">
            <!-- 數量 -->
            <div class="mb-3">
              <label class="form-label">數量</label>
              <input
                type="number"
                v-model.number="tempSelected.count"
                min="1"
                class="form-control"
              />
            </div>

            <!-- 冰塊 -->
            <div class="mb-3">
              <label class="form-label d-block">冰塊*</label>
              <div
                class="form-check form-check-inline"
                v-for="(ice, idx) in iceType"
                :key="'ice' + idx"
              >
                <input
                  class="form-check-input"
                  type="radio"
                  name="iceRadio"
                  :id="'ice' + idx"
                  :value="ice"
                  v-model="tempSelected.ice"
                  :disabled="isIceFixed(ice)"
                />
                <label class="form-check-label" :for="'ice' + idx">{{ ice }}</label>
              </div>
            </div>

            <!-- 甜度 -->
            <div class="mb-3">
              <label class="form-label d-block">甜度*</label>
              <div
                class="form-check form-check-inline"
                v-for="(sugar, idx) in sugarType"
                :key="'sugar' + idx"
              >
                <input
                  class="form-check-input"
                  type="radio"
                  name="sugarRadio"
                  :id="'sugar' + idx"
                  :value="sugar"
                  v-model="tempSelected.sugar"
                  :disabled="isSugarFixed(sugar)"
                />
                <label class="form-check-label" :for="'sugar' + idx">{{ sugar }}</label>
              </div>
            </div>

            <!-- 加料 -->
            <div class="mb-3">
              <label class="form-label d-block">加料</label>
              <div
                class="form-check form-check-inline"
                v-for="(top, idx) in toppingsType"
                :key="'top' + idx"
              >
                <input
                  class="form-check-input"
                  type="checkbox"
                  :id="'top' + idx"
                  :value="top"
                  v-model="tempSelected.toppings"
                  :disabled="isToppingFixed(top)"
                />
                <label class="form-check-label" :for="'top' + idx">{{ top }}</label>
              </div>
            </div>

            <!-- 備註 -->
            <div class="mb-3">
              <label class="form-label">備註</label>
              <textarea v-model.trim="tempSelected.notice" rows="2" class="form-control"></textarea>
            </div>

            <!-- 動作按鈕 -->
            <div class="d-flex gap-2">
              <button class="btn btn-outline-secondary w-100" @click="reset">取消</button>
              <button
                class="btn btn-primary w-100"
                :disabled="!tempSelected.name"
                @click="addToOrder"
              >
                加入
              </button>
            </div>
          </div>
        </div>

        <!-- ================= 訂單列表 ================= -->
        <div v-if="orderList.length" class="card mb-3">
          <div class="card-body">
            <table class="table align-middle">
              <thead>
                <tr>
                  <th>品項</th>
                  <th>冰</th>
                  <th>甜</th>
                  <th>加料</th>
                  <th>單價</th>
                  <th>數量</th>
                  <th class="text-end">小計</th>
                  <th></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(item, idx) in orderList" :key="'o-' + idx">
                  <td>
                    {{ item.name }}<br />
                    <small v-if="item.notice" class="text-muted">備註：{{ item.notice }}</small>
                  </td>
                  <td>{{ item.ice }}</td>
                  <td>{{ item.sugar }}</td>
                  <td>{{ item.toppings.toString() }}</td>
                  <td>{{ item.price + item.toppings.length * 10 }}</td>
                  <td>{{ item.count }}</td>
                  <td class="text-end">{{ item.total }}</td>
                  <td>
                    <button class="btn btn-sm btn-outline-danger" @click="removeOrder(idx)">
                      刪除
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>

            <p class="mb-1">品項數：{{ orderItemCount }} 杯</p>
            <p class="mb-1">加料費：NT$ {{ toppingFeeTotal }}</p>
            <p class="mb-1 fw-bold">總計：NT$ {{ orderTotal }} 元</p>

            <button
              class="btn btn-success w-100"
              :disabled="!orderList.length"
              @click="generateOrder"
            >
              產生訂單
            </button>
          </div>
        </div>

        <!-- ================= 已確認訂單 ================= -->
        <div v-if="checkedOrder.orders.length" class="card">
          <div class="card-body">
            <h5 class="mb-3">訂單明細</h5>
            <p class="mb-1">時間：{{ checkedOrder.time }}</p>
            <p class="mb-1">杯數：{{ checkedOrder.orders.length }}</p>
            <p class="mb-3">付款狀態：未付款</p>

            <table class="table">
              <thead>
                <tr>
                  <th>品項</th>
                  <th>冰</th>
                  <th>甜</th>
                  <th>加料</th>
                  <th>價格</th>
                  <th>數量</th>
                  <th class="text-end">小計</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(i, k) in checkedOrder.orders" :key="'c' + k">
                  <td>
                    {{ i.name }}<br />
                    <small v-if="i.notice" class="text-muted">備註：{{ i.notice }}</small>
                  </td>
                  <td>{{ i.ice }}</td>
                  <td>{{ i.sugar }}</td>
                  <td>{{ i.toppings.toString() }}</td>
                  <td>{{ i.price + i.toppings.length * 10 }}</td>
                  <td>{{ i.count }}</td>
                  <td class="text-end">{{ i.total }}</td>
                </tr>
              </tbody>
            </table>

            <p class="text-end fw-bold">共 NT$ {{ checkedOrder.total }} 元</p>
            <button class="btn btn-outline-primary w-100" @click="finishPayment">結帳</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      tempSelected: {},
      orderList: [],
      checkedOrder: {
        time: '',
        total: 0,
        orders: [],
      },
      iceType: ['正常冰', '少冰', '微冰', '去冰', '熱'],
      sugarType: ['全糖', '七分', '半糖', '三分', '無糖'],
      toppingsType: ['珍珠', '粉條', '小粉圓', '椰果', '芋頭'],
      products: [
        {
          name: '珍珠鮮奶茶',
          engName: 'Pearl Milk Tea',
          price: 60,
          defaults: { toppings: ['珍珠'], sugar: '', ice: '' },
        },
        {
          name: '椰果鮮奶茶',
          engName: 'Coconut Milk Tea',
          price: 60,
          defaults: { toppings: ['椰果'], sugar: '', ice: '' },
        },
        {
          name: '鮮奶茶',
          engName: 'Milk Tea',
          price: 50,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '古意冬瓜茶 (糖固定)',
          engName: 'Winter Melon Drink',
          price: 30,
          defaults: { toppings: [''], sugar: '全糖', ice: '' },
        },
        {
          name: '蜜香紅茶',
          engName: 'Black Tea',
          price: 30,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '包種青茶',
          engName: 'Baozhong Tea',
          price: 35,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '檸檬烏龍',
          engName: 'Lemon Oolong Tea',
          price: 55,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '薑母茶 (熱飲)',
          engName: 'Ginger Tea',
          price: 55,
          defaults: { toppings: [''], sugar: '', ice: '熱' },
        },
        {
          name: '青草茶',
          engName: 'Herbal Tea',
          price: 35,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '金桔檸檬',
          engName: 'Kumquat Lemonade',
          price: 40,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
        {
          name: '柳澄青茶',
          engName: 'Orange Mountain Tea',
          price: 45,
          defaults: { toppings: [''], sugar: '', ice: '' },
        },
      ],
    }
  },
  computed: {
    orderItemCount() {
      return this.orderList.reduce((sum, item) => sum + Number(item.count), 0)
    },
    orderTotal() {
      return this.orderList.reduce((sum, item) => sum + item.total, 0)
    },
    toppingFeeTotal() {
      return this.orderList.reduce((sum, item) => sum + item.toppings.length * 10 * item.count, 0)
    },
  },
  methods: {
    selectedProduct(product) {
      this.tempSelected = {
        ...product,
        count: 1,
        ice: product.defaults.ice || '正常冰',
        sugar: product.defaults.sugar || '全糖',
        toppings: [],
        notice: '',
      }
    },
    reset() {
      this.tempSelected = {}
    },
    isIceFixed(opt) {
      const d = this.tempSelected.defaults
      return d && d.ice !== '' && opt !== d.ice
    },
    isSugarFixed(opt) {
      const d = this.tempSelected.defaults
      return d && d.sugar !== '' && opt !== d.sugar
    },
    isToppingFixed(top) {
      const d = this.tempSelected.defaults
      return d && d.toppings && d.toppings.includes(top)
    },
    addToOrder() {
      if (!this.tempSelected.name) return
      const { price, toppings, count } = this.tempSelected
      const toppingFee = toppings.length * 10
      const total = (price + toppingFee) * count
      this.orderList.push({ ...this.tempSelected, total })
      this.reset()
      this.saveOrderList()
    },
    removeOrder(idx) {
      this.orderList.splice(idx, 1)
      this.saveOrderList()
    },
    generateOrder() {
      if (!this.orderList.length) return
      const date = new Date().toLocaleString()
      this.checkedOrder = {
        time: date,
        total: this.orderTotal,
        orders: JSON.parse(JSON.stringify(this.orderList)),
      }
      this.orderList = []
      this.reset()
      this.saveOrderList()
    },
    finishPayment() {
      this.checkedOrder = { time: '', total: 0, orders: [] }
      localStorage.removeItem('drink-orders')
    },
    saveOrderList() {
      localStorage.setItem('drink-orders', JSON.stringify(this.orderList))
    },
    loadOrderList() {
      const saved = localStorage.getItem('drink-orders')
      if (saved) this.orderList = JSON.parse(saved)
    },
  },
  mounted() {
    this.loadOrderList()
  },
}
</script>

<style scoped></style>
