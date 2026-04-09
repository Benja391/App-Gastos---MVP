<template>
  <section class="flex items-center justify-center min-h-screen bg-[#08a04b] text-white px-4 py-10 mt-8">
    <div
      class="relative bg-white/95 backdrop-blur-sm text-gray-900 rounded-3xl border border-white/20 w-full max-w-6xl p-6 md:p-8 mt-1 overflow-hidden shadow-[0_8px_30px_rgba(0,0,0,0.35)] "
    >
      <BaseHeading>Historial</BaseHeading>
      <p class="text-center text-gray-600 mb-6">Consultá tus movimientos sin mezclar gastos.</p>

      <!-- ❌ TABS (dejamos solo gastos) -->
      <!--
      <div class="flex justify-center mb-6">
        <div class="inline-flex bg-gray-100 rounded-2xl p-1 shadow-sm">
          <button @click="activeTab = 'gastos'">Gastos</button>
          <button @click="activeTab = 'ingresos'">Ingresos</button>
        </div>
      </div>
      -->

      <!-- ✅ GASTOS -->
      <div class="space-y-4">

        <button
          type="button"
          @click="showFilters = !showFilters"
          class="w-full md:w-auto px-4 py-2 rounded-xl border border-green-200 text-green-700 font-semibold hover:bg-green-50 transition"
        >
          {{ showFilters ? 'Ocultar filtros' : 'Mostrar filtros' }}
        </button>

        <!-- FILTROS -->
        <div v-if="showFilters" class="bg-gray-50 border border-gray-200 rounded-2xl p-4 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
          
          <div>
            <label class="text-xs font-semibold text-gray-600">Fecha</label>
            <select v-model="filters.dateRange" class="mt-1 w-full px-3 py-2 rounded-lg border">
              <option value="week">Última semana</option>
              <option value="month">Último mes</option>
              <option value="year">Último año</option>
              <option value="custom">Personalizado</option>
            </select>
          </div>

          <div>
            <label class="text-xs font-semibold text-gray-600">Categoría</label>
            <select v-model="filters.category" class="mt-1 w-full px-3 py-2 rounded-lg border">
              <option value="">Todas</option>
              <option v-for="cat in uniqueCategories" :key="cat">{{ cat }}</option>
            </select>
          </div>

          <div>
            <label class="text-xs font-semibold text-gray-600">Monto</label>
            <select v-model="filters.amountOperator" class="mt-1 w-full px-3 py-2 rounded-lg border">
              <option value="">Sin filtro</option>
              <option value=">">Mayor a</option>
              <option value="<">Menor a</option>
            </select>
          </div>

          <div>
            <label class="text-xs font-semibold text-gray-600">Valor</label>
            <input v-model.number="filters.amountValue" type="number" class="mt-1 w-full px-3 py-2 rounded-lg border" />
          </div>

        </div>

        <!-- LISTA -->
        <div v-if="paginatedExpenses.length" class="grid gap-3 mt-3 max-h-[620px] overflow-y-auto pr-2">
          <div v-for="exp in paginatedExpenses" :key="exp.id" class="bg-white border p-4 rounded-xl">

            <h3 class="font-bold">🧾 {{ exp.name }}</h3>
            <p>{{ exp.category }}</p>
            <p class="text-red-600">{{ formatCurrency(exp.amount) }}</p>

          </div>
        </div>

        <p v-else class="text-center text-gray-500 mt-6 text-sm">
          No se encontraron gastos.
        </p>
      </div>

      <!-- ❌ INGRESOS COMPLETAMENTE COMENTADOS -->
      <!--
      <div v-else class="space-y-4">
        <div class="bg-green-50 border border-green-200 p-5 rounded-xl">
          <p>Saldo inicial</p>
          <p>{{ formatCurrency(saldos.initialAmount) }}</p>
        </div>

        <div>
          <h3>Ingresos registrados</h3>
          <div v-for="item in saldosList">
            {{ item.amount }}
          </div>
        </div>
      </div>
      -->

    </div>
  </section>
</template>

<script>
import { getAuth, onAuthStateChanged } from "firebase/auth";
import { getFirestore, collection, getDocs, query, where, doc, getDoc, deleteDoc, updateDoc } from "firebase/firestore";
import BaseHeading from "../components/BaseHeading.vue";

export default {
  components: { BaseHeading },

  data() {
    return {
      db: getFirestore(),
      currentUserUid: null,

      // ❌ TAB eliminado visualmente
      // activeTab: "gastos",

      showFilters: false,
      expenses: [],
      currentPage: 1,
      itemsPerPage: 8,

      // ❌ INGRESOS (comentado)
      /*
      saldos: {
        initialAmount: 0,
        remainingAmount: 0,
        additionalAmounts: [],
      },
      saldosList: [],
      */

      filters: {
        dateRange: "month",
        category: "",
        amountOperator: "",
        amountValue: null,
      },
    };
  },

  computed: {
    filteredExpenses() {
      return this.expenses;
    },

    totalPages() {
      return Math.ceil(this.filteredExpenses.length / this.itemsPerPage) || 1;
    },

    paginatedExpenses() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      return this.filteredExpenses.slice(start, start + this.itemsPerPage);
    },

    uniqueCategories() {
      return [...new Set(this.expenses.map((e) => e.category).filter(Boolean))];
    },
  },

  methods: {
    async fetchExpenses(uid) {
      const q = query(collection(this.db, "gastos"), where("uid", "==", uid));
      const snapshot = await getDocs(q);
      this.expenses = snapshot.docs.map((d) => ({ id: d.id, ...d.data() }));
    },

    // ❌ INGRESOS (comentado)
    /*
    async fetchSaldos(uid) {
      ...
    }
    */

    async deleteExpense(exp) {
      await deleteDoc(doc(this.db, "gastos", exp.id));
      this.expenses = this.expenses.filter((e) => e.id !== exp.id);
    },

    formatCurrency(value) {
      return new Intl.NumberFormat("es-AR", {
        style: "currency",
        currency: "ARS",
      }).format(value);
    },
  },

  mounted() {
    const auth = getAuth();
    onAuthStateChanged(auth, async (user) => {
      if (!user) return;
      this.currentUserUid = user.uid;

      await this.fetchExpenses(user.uid);

      // ❌ INGRESOS (comentado)
      // await this.fetchSaldos(user.uid);
    });
  },
};
</script>