<script setup>
import { BrowserQRCodeReader } from "@zxing/browser";
import { ref, onMounted, onBeforeUnmount, nextTick, watch } from "vue";
import axios from "axios";

const API_BASE = "https://tbn4smf2-3000.asse.devtunnels.ms";

const result = ref("");
const status = ref("");

const isPromptOpen = ref(false);
const pendingId = ref("");
const lastActivity = ref({});
const EmployeeName = ref("");
const EmployeeId = ref("");
const EmployeeDivisi = ref("");
const promptSession = ref(0); // force remount modal
const selectedKeterangan = ref(null);
const keteranganOptions = ["Toilet", "Makan", "Sholat", "Istirahat", "Lainnya"];
const lainnyaInput = ref(null);

const reader = new BrowserQRCodeReader();
let controls = null;
let scanningLocked = false; // cegah re-entrancy
const devices = ref([]);
const selectedDeviceId = ref(null);
// Ambil daftar kamera
const loadDevices = async () => {
  try {
    const videoInputDevices = await BrowserQRCodeReader.listVideoInputDevices();
    devices.value = videoInputDevices;
    // Default: kamera pertama
    if (videoInputDevices.length > 0) {
      selectedDeviceId.value = videoInputDevices[0].deviceId;
    }
  } catch (err) {
    console.error("Gagal ambil kamera", err);
  }
};

// ===== HTTP helpers =====
const checkActivity = async (id) => {
  // GET /activity/check/:id  -> { active: boolean, ... }
  const { data } = await axios.get(
    `${API_BASE}/activity/check/${encodeURIComponent(id)}`
  );
  // console.log(data);
  EmployeeId.value = data.karyawan_id;
  EmployeeName.value = data.nama;
  EmployeeDivisi.value = data.divisi;

  return data; // expected: { active: boolean, ... }
};

const postActivityIn = async (id) => {
  const { data } = await axios.post(`${API_BASE}/activity/in`, {
    karyawan_id: id,
  });
  getLastActivity();
  return data;
};

const getLastActivity = async () => {
  const { data } = await axios.get(`${API_BASE}/activity/last`);
  // console.log(data);

  lastActivity.value = data;
  return data;
};

onMounted(async () => {
  await loadDevices();
  startScanning();
});

onBeforeUnmount(() => {
  stopScanning();
});

const switchCamera = async (deviceId) => {
  stopScanning();
  selectedDeviceId.value = deviceId;
  await nextTick();
  startScanning();
};

const postActivityOut = async (id, jenis) => {
  const { data } = await axios.post(`${API_BASE}/activity/out`, {
    karyawan_id: id,
    jenis,
  });
  getLastActivity();
  return data;
};

// ===== Camera controls =====
// Start scanning dengan kamera terpilih
const startScanning = async () => {
  try {
    const previewElem = document.getElementById("preview");
    controls = await reader.decodeFromVideoDevice(
      selectedDeviceId.value, // gunakan kamera sesuai pilihan
      previewElem,
      handleScan
    );
  } catch (e) {
    console.error(e);
    status.value = "Gagal mengakses kamera.";
  }
};

const stopScanning = () => {
  if (controls) {
    controls.stop();
    controls = null;
  }
};

// ===== Modal helpers =====
const openPrompt = async (id) => {
  pendingId.value = id;
  selectedKeterangan.value = null;
  promptSession.value++; // paksa remount modal & radio
  isPromptOpen.value = true;
  await nextTick();
};

const closePrompt = () => {
  isPromptOpen.value = false;
};

// ===== Scan handler =====
const handleScan = async (resultObj, err) => {
  if (err || scanningLocked || isPromptOpen.value) return;
  if (!resultObj) return;

  const text = resultObj.getText?.() || "";
  if (!text) return;

  scanningLocked = true;
  stopScanning();

  try {
    const check = await checkActivity(text);
    const isActive = !!check?.active;

    if (isActive) {
      // sudah aktif → kirim IN
      const res = await postActivityIn(text);
      result.value = `${text} (kembali)`;
      status.value = res?.message || "Kembali berhasil";
      startScanning();
    } else {
      // belum aktif → buka modal untuk pilih jenis (OUT)
      await openPrompt(text);
    }
  } catch (e) {
    console.error(e);
    status.value = e?.response?.data?.message || e.message;
    startScanning();
  } finally {
    scanningLocked = false;
  }
};

// ===== Submit saat pilih keterangan =====
const autoConfirm = async (ket) => {
  if (!pendingId.value || !ket) return;
  try {
    const res = await postActivityOut(pendingId.value, ket);
    result.value = `${pendingId.value} (keluar)`;
    status.value = res?.message || `Keluar dengan keterangan: ${ket}`;
    // console.log(res);
  } catch (e) {
    console.error(e);
    status.value = e?.response?.data?.message || e.message;
  } finally {
    pendingId.value = "";
    closePrompt();
    startScanning();
  }
};

// Watcher: radio berubah → auto kirim (kecuali 'Lainnya')
watch(selectedKeterangan, async (val) => {
  if (!isPromptOpen.value) return;
  if (val === "Lainnya") {
    await nextTick();
    lainnyaInput.value?.focus();
    return;
  }
  if (val) await autoConfirm(val);
});

const confirmLainnyaIfReady = async (e) => {
  const val = (e?.target?.value || "").trim();
  if (val) {
    selectedKeterangan.value = val; // biar konsisten di UI
    await autoConfirm(val);
  }
};

// ===== Keyboard ESC untuk batal =====
const handleEsc = (e) => {
  if (e.key === "Escape" && isPromptOpen.value) {
    // batal pilih → lanjut scan lagi tanpa kirim
    closePrompt();
    pendingId.value = "";
    startScanning();
  }
};

onMounted(() => {
  startScanning();
  window.addEventListener("keydown", handleEsc);
});

onBeforeUnmount(() => {
  stopScanning();
  window.removeEventListener("keydown", handleEsc);
});
</script>

<template>
  <div class="p-4 flex flex-col items-center">
    <h1 class="text-xl font-bold mb-2">Scan QR Karyawan</h1>

    <video id="preview" class="w-full max-w-md border rounded-lg"></video>

    <!-- Dropdown untuk pilih kamera -->
    <div class="mt-4">
      <label>Pilih Kamera:</label>
      <select
        v-model="selectedDeviceId"
        @change="switchCamera(selectedDeviceId)"
        class="border p-1 rounded"
      >
        <option
          v-for="dev in devices"
          :key="dev.deviceId"
          :value="dev.deviceId"
        >
          {{ dev.label || "Kamera " + dev.deviceId }}
        </option>
      </select>
    </div>

    <!-- table last activity  -->

    <p class="text-green-600 font-semibold">{{ status }}</p>

    <div>Last Activity</div>

    <table class="w-1/2 text-sm text-left rtl:text-right text-gray-500">
      <thead class="text-xs text-gray-700 uppercase bg-gray-50">
        <tr>
          <th>Nama</th>
          <th>Divisi</th>
          <th>Keterangan</th>
          <th>Jam Keluar</th>
          <th>Jam Masuk</th>
          <th>Durasi</th>
        </tr>
      </thead>

      <tbody class="bg-white border-b border-gray-200">
        <tr v-for="item in lastActivity" :key="item.id">
          <td>{{ item.nama }}</td>
          <td>{{ item.divisi }}</td>
          <td>{{ item.jenis_aktivitas }}</td>
          <td>{{ item.jam_keluar }}</td>
          <td>{{ item.jam_masuk }}</td>
          <td>{{ item.durasi }}</td>
        </tr>
      </tbody>
    </table>
    <!-- Modal: key supaya remount bersih setiap buka -->
    <div
      v-if="isPromptOpen"
      :key="promptSession"
      class="fixed inset-0 z-50 flex items-center justify-center"
      aria-modal="true"
      role="dialog"
    >
      <!-- Backdrop -->
      <div class="absolute inset-0 bg-black/50"></div>

      <!-- Card -->
      <div
        class="relative w-full max-w-md mx-4 rounded-xl bg-white shadow-lg p-5"
      >
        <h2 class="text-lg font-semibold mb-3 text-gray-800">
          Pilih Keterangan (otomatis kirim)
        </h2>

        <div class="mb-4 text-sm text-gray-600">
          ID terdeteksi:
          <span class="font-mono font-medium text-gray-800"
            >{{ EmployeeId }}
          </span>
        </div>
        <div class="mb-4 text-sm text-gray-600">
          Nama
          <span class="font-mono font-medium text-gray-800"
            >{{ EmployeeName }}
          </span>
        </div>
        <div class="mb-4 text-sm text-gray-600">
          Divisi
          <span class="font-mono font-medium text-gray-800"
            >{{ EmployeeDivisi }}
          </span>
        </div>

        <div class="space-y-2 max-h-60 overflow-auto">
          <label
            v-for="opt in keteranganOptions"
            :key="opt"
            class="flex items-center gap-3 p-2 rounded-lg border hover:bg-gray-50 cursor-pointer"
          >
            <input
              type="radio"
              class="accent-black"
              name="keterangan"
              :value="opt"
              v-model="selectedKeterangan"
            />
            <span class="text-sm text-gray-800">{{ opt }}</span>
          </label>

          <!-- Input 'Lainnya' -->
          <div v-if="selectedKeterangan === 'Lainnya'" class="mt-2">
            <input
              ref="lainnyaInput"
              type="text"
              class="w-full border rounded-lg p-2"
              placeholder="Tuliskan keterangan lalu Enter"
              @keyup.enter="confirmLainnyaIfReady"
              @blur="confirmLainnyaIfReady"
            />
            <p class="text-xs text-gray-500 mt-1">
              Tekan Enter untuk mengirim.
            </p>
          </div>
        </div>

        <div class="mt-4 text-right">
          <button
            class="px-3 py-1 rounded-lg border hover:bg-gray-50"
            @click="closePrompt"
          >
            Tutup
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
