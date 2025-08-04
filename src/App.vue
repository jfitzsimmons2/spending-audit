<script setup lang="ts">
import Papa from 'papaparse';
import { ref } from 'vue';
import TransactionCategorize from './components/TransactionCategorize.vue';

const fileInput = ref<HTMLInputElement>();

// const config = {
// 	delimiter: "",	// auto-detect
// 	newline: "",	// auto-detect
// 	quoteChar: '"',
// 	escapeChar: '"',
// 	header: false,
// 	transformHeader: undefined,
// 	dynamicTyping: false,
// 	preview: 0,
// 	encoding: "",
// 	worker: false,
// 	comments: false,
// 	step: undefined,
// 	complete: undefined,
// 	error: undefined,
// 	download: false,
// 	downloadRequestHeaders: undefined,
// 	downloadRequestBody: undefined,
// 	skipEmptyLines: false,
// 	chunk: undefined,
// 	chunkSize: undefined,
// 	fastMode: undefined,
// 	beforeFirstChunk: undefined,
// 	withCredentials: undefined,
// 	transform: undefined,
// 	delimitersToGuess: [',', '\t', '|', ';', Papa.RECORD_SEP, Papa.UNIT_SEP],
// 	skipFirstNLines: 0

// }

const parsedData = ref([] as any[]);
const parsingError = ref();
const headers = ref();
const loading = ref(false);

const handleChange = (_event: any) => {
    loading.value = true;
    if (!fileInput.value || !fileInput.value.files) {
        return;
    }
    const file = fileInput.value.files[0];
    console.log('file', file);
    if (!file) {
        return;
    }

    // Clear previous data
    // parsedData.value = null;
    parsingError.value = null;
    headers.value = [];

    Papa.parse(file, {
        header: true, // Set to true if your CSV has a header row
        skipEmptyLines: true,
        complete: (results: any) => {
            // The `results` object contains the parsed data
            // results.data is the array of rows
            // results.meta.fields is the array of headers
            console.log('Parsed results:', results);
            if (results.errors.length) {
                parsingError.value = results.errors[0].message;
            } else {
                headers.value = results.meta.fields;
                parsedData.value.push(...results.data.filter((row: any) => row.Type === 'Sale').map((row: any) => {
                    // Convert Amount to a number
                    if (row.Amount) {
                        row.Amount = Math.abs(parseFloat(row.Amount.replace(/[^0-9.-]+/g, '')));
                    }

                    row.Source = file.name;

                    return row;
                }));
            }
        },
        error: (err: any) => {
            // This is a more general error handler for file loading issues
            console.error('File parsing error:', err);
            parsingError.value = 'An error occurred during file parsing.';
        }
    });
    loading.value = false;
}

</script>

<template>
    <div class="flex flex-col gap-8 p-8 max-w-4xl">
        <div class="flex items-center justify-between gap-4">
            <p>To start, upload a CSV export from your Chase.com account</p>
            <div>
                <input @change="handleChange" ref="fileInput" type="file" id="file" name="file" />
                <label for="file">Choose a file</label>
            </div>
        </div>
        <div>
            <TransactionCategorize v-if="parsedData && parsedData.length" :transactions="parsedData" />
        </div>
    </div>
</template>
