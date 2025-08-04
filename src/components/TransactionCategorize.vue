<script setup lang="ts">
import { computed, ref } from 'vue'
import { onMounted, onUnmounted } from 'vue'
import Papa from 'papaparse';

const props = defineProps<{ transactions: any[] }>()
const transactions = ref(props.transactions);
const categories = ref(['Bills & utilities', 'Groceries', 'Entertainment', 'Food & drink', 'Shopping', 'Home', 'Health', 'Professional services', 'Gas']);
const ci = ref(0) // current index
const ct = computed({
    get() { return transactions.value[ci.value] },
    set(value) {
        transactions.value[ci.value] = value
    }
}) // current transaction

const getNextUncategorizedTransaction = () => {
    return transactions.value.findIndex((t) => !Object.keys(t).includes('NecessityLevel'));
}

const handleKeydown = (e: KeyboardEvent) => {
    if (e.key === 'a') {
        ct.value = { ...ct.value, "NecessityLevel": 'Necessary' }
    } else if (e.key === 's') {
        ct.value = { ...ct.value, "NecessityLevel": 'Necessary, but reviewable' }
    } else if (e.key === 'd') {
        ct.value = { ...ct.value, "NecessityLevel": 'Unnecessary' }
    } else if (e.key === 'ArrowLeft') {
        if (ci.value > 0) {
            ci.value--;
        }
    } else if (e.key === 'ArrowRight') {
        if (ci.value < transactions.value.length - 1) {
            ci.value++;
        }
    } else if (e.key === 'Enter') {
        // Move to the next transaction
        const nextIndex = getNextUncategorizedTransaction();
        if (nextIndex !== -1) {
            ci.value = nextIndex;
        }

    } else if (e.key === 'ArrowUp') {
        // Find current category index
        const currentCategoryIndex = categories.value.indexOf(ct.value['Category']);
        // Move to the previous category, wrapping around if necessary
        const newCategoryIndex = (currentCategoryIndex - 1 + categories.value.length) % categories.value.length;
        ct.value['Category'] = categories.value[newCategoryIndex];
    } else if (e.key === 'ArrowDown') {
        // Find current category index
        const currentCategoryIndex = categories.value.indexOf(ct.value['Category']);
        // Move to the next category, wrapping around if necessary
        const newCategoryIndex = (currentCategoryIndex + 1) % categories.value.length;
        ct.value['Category'] = categories.value[newCategoryIndex];
    }
}

onMounted(() => {
    window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
    window.removeEventListener('keydown', handleKeydown)
})

const applyToAll = computed(() => {
    return {
        transactions: transactions.value.every((t) => t['Description'] === ct.value['Description'] && !Object.keys(t).includes('NecessityLevel')),
        quantity: transactions.value.filter((t) => t['Description'] === ct.value['Description']).length,
    }
})

const applyToAllAction = () => {
    transactions.value.forEach((t) => {
        if (ct.value['Description'] === t['Description']) {
            t['NecessityLevel'] = ct.value['NecessityLevel'];
            t['Category'] = ct.value['Category'];
        }
    });
}

// Function to export JSON data to a CSV file and download it
const exportToCsv = (filename: string) => {
    // 1. Convert the JSON object to a CSV string
    const csvString = Papa.unparse(transactions.value);

    // 2. Create a Blob from the CSV string
    const blob = new Blob([csvString], { type: 'text/csv;charset=utf-8;' });

    // 3. Create a temporary download link
    const link = document.createElement("a");

    if (link.download !== undefined) { // Check for download attribute support
        const url = URL.createObjectURL(blob);
        link.setAttribute("href", url);
        link.setAttribute("download", filename);
        link.style.visibility = 'hidden';

        // 4. Add the link to the document, click it, and then remove it
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);

        // 5. Clean up the URL object
        window.URL.revokeObjectURL(url);
    } else {
        // Fallback for older browsers
        alert('Your browser does not support downloading files directly. Please copy and paste the CSV content.');
    }
}

const getIcon = (necessityLevel: string) => {
    switch (necessityLevel) {
        case 'Necessary':
            return '✅';
        case 'Necessary, but reviewable':
            return '⚠️';
        case 'Unnecessary':
            return '❌';
        default:
            return '';
    }
}

const getTotal = (necessityLevel: string) => {
    return Intl.NumberFormat().format(
        transactions.value
            .filter((t) => t['NecessityLevel'] === necessityLevel)
            .reduce((sum, t) => sum + Number(t['Amount'] || 0), 0)
    );
}

const remainingTransactions = computed(() => {
    return transactions.value.filter((t) => !Object.keys(t).includes('NecessityLevel'));
});
</script>

<template>
    <div class="flex flex-col gap-8 max-w-4xl">
        <div class="flex gap-8 justify-between items-center">
            <strong>{{ ci + 1 }} / {{ transactions.length }} transactions ({{ remainingTransactions.length }}
                remaining)</strong>
            <button @click="getNextUncategorizedTransaction">Find next uncategorized transaction</button>

            <div class="flex gap-4">
                <div>
                    {{transactions.filter((t) => t['NecessityLevel'] === 'Necessary').length}}
                    Necessary<br />
                    {{ getTotal('Necessary') }}
                </div>
                <div>
                    {{transactions.filter((t) => t['NecessityLevel'] === 'Necessary, but reviewable').length}}
                    Reviewable<br />
                    {{ getTotal('Necessary, but reviewable') }}
                </div>
                <div>
                    {{transactions.filter((t) => t['NecessityLevel'] === 'Unnecessary').length}}
                    Unnecessary<br />
                    {{ getTotal('Unnecessary') }}
                </div>
            </div>
        </div>

        <div class="border b-blue b-solid p-2 flex gap-4 max-w-4xl justify-between">
            <div class="basis-100px">{{ ct["Transaction Date"] }}</div>
            <div class="flex-1">{{ ct["Description"] }}</div>
            <div class="basis-100px">{{ ct["Amount"] }}</div>
            <select v-model="ct['Category']">
                <option v-for="(category, index) in categories" :key="index" :value="category">{{ category }}</option>
            </select>
            <span class="basis-250px">{{ getIcon(ct['NecessityLevel']) }} {{ ct['NecessityLevel'] }}</span>
            <div v-if="applyToAll.quantity > 1" class="flex gap-2">
                <button @click="applyToAllAction">Apply to all {{ applyToAll.quantity }} transactions</button>

            </div>
        </div>

        <div class="flex gap-2 justify-end items-center"
            v-if="transactions.every((t) => Object.keys(t).includes('NecessityLevel'))">
            <strong>All necessity levels categorized</strong>
            <button @click="exportToCsv('transactions.csv')">Export</button>
        </div>


    </div>

</template>
