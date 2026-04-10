<template>
  <section class="flex items-center justify-center min-h-screen bg-[#08a04b] text-white px-4 py-10 mt-8">
    <div class="relative bg-white/95 text-gray-900 rounded-3xl w-full max-w-6xl p-6 md:p-8 shadow-[0_8px_30px_rgba(0,0,0,0.35)]">

      <BaseHeading>Historial</BaseHeading>
      <p class="text-center text-gray-600 mb-6">Consultá tus movimientos sin mezclar gastos.</p>

      <div class="space-y-4">

        <!-- BOTONES -->
        <div class="flex gap-3 flex-wrap">
          <button @click="showFilters = !showFilters"
            class="px-4 py-2 rounded-xl border border-green-200 text-green-700 font-semibold hover:bg-green-50">
            {{ showFilters ? 'Ocultar filtros' : 'Mostrar filtros' }}
          </button>

          <button @click="clearFilters"
            class="px-4 py-2 rounded-xl bg-gray-200 text-gray-700 hover:bg-gray-300">
            Limpiar filtros
          </button>
        </div>

        <!-- FILTROS -->
        <div v-if="showFilters" class="bg-gray-50 border rounded-2xl p-4 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">

          <!-- FECHA -->
          <div>
            <label class="text-xs font-semibold text-gray-600">Fecha</label>
            <select v-model="filters.dateRange"
              class="mt-1 w-full px-3 py-2 rounded-lg border h-[42px]">
              <option value="week">Última semana</option>
              <option value="month">Último mes</option>
              <option value="year">Último año</option>
              <option value="custom">Personalizado</option>
            </select>
          </div>

          <!-- PERSONALIZADO -->
         <div v-if="filters.dateRange === 'custom'" class="col-span-2 flex gap-2 items-end">
           <input v-model="filters.startDate" type="date"
  class="w-full px-3 h-[42px] rounded-lg border appearance-none" />

             <input v-model="filters.endDate" type="date"
  class="w-full px-3 h-[42px] rounded-lg border appearance-none" />
          </div>

          <!-- CATEGORÍA -->
          <div>
            <label class="text-xs font-semibold text-gray-600">Categoría</label>
            <select v-model="filters.category"
              class="mt-1 w-full px-3 py-2 rounded-lg border h-[42px]">
              <option value="">Todas</option>
              <option v-for="cat in uniqueCategories" :key="cat">{{ cat }}</option>
            </select>
          </div>

          <!-- MONTO -->
          <div>
            <label class="text-xs font-semibold text-gray-600">Monto</label>
            <select v-model="filters.amountOperator"
              class="mt-1 w-full px-3 py-2 rounded-lg border h-[42px]">
              <option value="">Sin filtro</option>
              <option value=">">Mayor a</option>
              <option value="<">Menor a</option>
            </select>
          </div>

          <!-- VALOR -->
          <div>
            <label class="text-xs font-semibold text-gray-600">Valor</label>
            <input v-model.number="filters.amountValue" type="number"
              class="mt-1 w-full px-3 py-2 rounded-lg border h-[42px]" />
          </div>

        </div>

        <!-- LISTA -->
        <div v-if="paginatedExpenses.length"
          class="grid gap-3 mt-3 max-h-[420px] overflow-y-auto pr-2">

          <div v-for="exp in paginatedExpenses" :key="exp.id"
            class="bg-white border p-4 rounded-xl">

            <h3 class="font-bold">{{ exp.name }}</h3>
            <p class="text-gray-600">{{ exp.category }}</p>

            <!-- 🗓 FECHA -->
            <p class="text-xs text-gray-500 mt-1">
              {{ formatDate(exp.createdAt) }}
            </p>

            <!-- 💰 MONTO -->
            <p class="text-red-600 font-semibold mt-1">
              {{ formatCurrency(exp.amount) }}
            </p>

          </div>
        </div>

        <!-- MENSAJE -->
        <p v-else class="text-center text-gray-500 mt-6 text-sm">
          {{ noResultsMessage }}
        </p>

      </div>
    </div>
  </section>
</template>

<script>
import { getAuth, onAuthStateChanged } from "firebase/auth";
import { getFirestore, collection, getDocs, query, where, orderBy } from "firebase/firestore";
import BaseHeading from "../components/BaseHeading.vue";

export default {
  components: { BaseHeading },

  data() {
    return {
      db: getFirestore(),
      currentUserUid: null,
      showFilters: false,
      expenses: [],
      currentPage: 1,
      itemsPerPage: 8,

      filters: {
        dateRange: "month",
        category: "",
        amountOperator: "",
        amountValue: null,
        startDate: null,
        endDate: null,
      },
    };
  },

  computed: {
    filteredExpenses() {
      let result = [...this.expenses];
      const now = new Date();

      // 📅 FILTROS NORMALES
      if (this.filters.dateRange === "week") {
        const lastWeek = new Date();
        lastWeek.setDate(now.getDate() - 7);
        result = result.filter(e => this.getDate(e) >= lastWeek);
      }

      if (this.filters.dateRange === "month") {
        const lastMonth = new Date();
        lastMonth.setMonth(now.getMonth() - 1);
        result = result.filter(e => this.getDate(e) >= lastMonth);
      }

      if (this.filters.dateRange === "year") {
        const lastYear = new Date();
        lastYear.setFullYear(now.getFullYear() - 1);
        result = result.filter(e => this.getDate(e) >= lastYear);
      }

      // 🔥 PERSONALIZADO (FIX REAL)
      if (this.filters.dateRange === "custom" && this.filters.startDate && this.filters.endDate) {
  const start = new Date(this.filters.startDate + "T00:00:00");
  const end = new Date(this.filters.endDate + "T23:59:59");

  result = result.filter(e => {
    const date = this.getDate(e);
    return date >= start && date <= end;
  });
}

      // CATEGORÍA
      if (this.filters.category) {
        result = result.filter(e => e.category === this.filters.category);
      }

      // MONTO
      if (this.filters.amountOperator && this.filters.amountValue !== null) {
        if (this.filters.amountOperator === ">") {
          result = result.filter(e => e.amount > this.filters.amountValue);
        }
        if (this.filters.amountOperator === "<") {
          result = result.filter(e => e.amount < this.filters.amountValue);
        }
      }

      return result.sort((a, b) => this.getDate(b) - this.getDate(a));
    },

    paginatedExpenses() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      return this.filteredExpenses.slice(start, start + this.itemsPerPage);
    },

    uniqueCategories() {
      return [...new Set(this.expenses.map(e => e.category).filter(Boolean))];
    },

    noResultsMessage() {
      if (!this.expenses.length) return "Todavía no hay gastos cargados.";

      if (!this.filteredExpenses.length) {
        return "No se encontraron gastos con esos filtros.";
      }

      return "";
    }
  },

  methods: {
    getDate(e) {
      return new Date(e.createdAt?.toDate?.() || e.createdAt);
    },

    async fetchExpenses(uid) {
      const q = query(collection(this.db, "gastos"), where("uid", "==", uid), orderBy("createdAt", "desc"));
      const snap = await getDocs(q);
      this.expenses = snap.docs.map(d => ({ id: d.id, ...d.data() }));
    },

    clearFilters() {
      this.filters = {
        dateRange: "month",
        category: "",
        amountOperator: "",
        amountValue: null,
        startDate: null,
        endDate: null,
      };
    },

    formatCurrency(value) {
      return new Intl.NumberFormat("es-AR", {
        style: "currency",
        currency: "ARS",
        minimumFractionDigits: 0,
        maximumFractionDigits: 0,
      }).format(value).replace(/\s/g, "");
    },

    formatDate(value) {
      const date = new Date(value?.toDate?.() || value);
      return date.toLocaleDateString("es-AR");
    }
  },

  mounted() {
    onAuthStateChanged(getAuth(), async user => {
      if (!user) return;
      this.currentUserUid = user.uid;
      await this.fetchExpenses(user.uid);
    });
  }
};
</script>