<!-- eslint-disable @typescript-eslint/no-explicit-any -->
<script setup lang="ts">
import { useJobSheetStore } from '@/stores/jobSheetStore'
import Swal from 'sweetalert2'
import { onMounted, ref, watch, computed, type ComponentPublicInstance } from 'vue'
import { useRoute } from 'vue-router'
import { VueDatePicker } from '@vuepic/vue-datepicker'
import '@vuepic/vue-datepicker/dist/main.css'
import Vue3Signature from 'vue3-signature'
import { PDFDocument, StandardFonts, rgb } from 'pdf-lib'
import AddSparePartModal from '@/components/Modals/AddSparePartModal.vue'
import AddLaserDataModal from '@/components/Modals/AddLaserDataModal.vue'
import logoUrl from '@/assets/logo.png'
const jobSheetStore = useJobSheetStore()
const loading = ref(true)
const uploading = ref(false)

const addSparePartOpen = ref(false)
const addLaserDataOpen = ref(false)

type Vue3SignatureExposed = {
  clear: () => void
  save: () => string // returns base64 png data url
}

const signaturePad = ref<ComponentPublicInstance<Vue3SignatureExposed> | null>(null)
const clearSignature = () => {
  signaturePad.value?.clear()
  if (jobSheetStore.jobSheetDetail) jobSheetStore.jobSheetDetail.customerSignature = null as any
}

const saveSignature = () => {
  const dataUrl = signaturePad.value?.save()

  if (!dataUrl) {
    Swal.fire('Empty signature', 'Please sign before saving.', 'warning')
    return
  }

  if (jobSheetStore.jobSheetDetail) {
    jobSheetStore.jobSheetDetail.customerSignature = dataUrl as any
  }

  Swal.fire('Saved', 'Signature captured successfully.', 'success')
}

async function uploadFile(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.item(0)

  if (!file) return
  if (!jobSheetStore.jobSheetDetail?.id) return

  uploading.value = true
  await jobSheetStore.uploadAttachment(file, jobSheetStore.jobSheetDetail.id)
  uploading.value = false
  input.value = ''
}

// PDF Generator
async function generateServicePDF() {
  const sheet = jobSheetStore.jobSheetDetail
  if (!sheet || !sheet.createdAt) return

  const pdfDoc = await PDFDocument.create()
  const page = pdfDoc.addPage([595.28, 841.89]) // A4

  const font = await pdfDoc.embedFont(StandardFonts.Helvetica)
  const bold = await pdfDoc.embedFont(StandardFonts.HelveticaBold)

  const { width, height } = page.getSize()

  let y = height - 40

  const safeY = (val: number) => (Number.isFinite(val) ? val : 40)

  function text(t: string, x: number, yy: number, size = 10, b = false) {
    const drawY = safeY(yy)
    page.drawText(t ?? '-', {
      x,
      y: drawY,
      size,
      font: b ? bold : font,
      color: rgb(0, 0, 0),
    })
  }

  function box(x: number, yy: number, w: number, h: number) {
    const drawY = safeY(yy)
    page.drawRectangle({
      x,
      y: drawY,
      width: w,
      height: h,
      borderWidth: 1,
      borderColor: rgb(0, 0, 0),
    })
  }

  function row(cols: string[], xs: number[], yy: number) {
    const rowHeight = 16
    const baseY = safeY(yy)

    cols.forEach((c, i) => text(c, (xs[i] || 0) + 4, baseY - 12, 9))

    page.drawLine({
      start: { x: 40, y: baseY - rowHeight },
      end: { x: width - 40, y: baseY - rowHeight },
      thickness: 1,
      color: rgb(0, 0, 0),
    })

    return baseY - rowHeight
  }
  // ── Add these two helpers near your other helpers (text, box, row) ──

  function wrapText(t: string, maxWidth: number, size: number): string[] {
    // First, split on any newline variants (\r\n, \r, \n)
    const paragraphs = (t ?? '-').split(/\r?\n|\r/)
    const lines: string[] = []

    for (const paragraph of paragraphs) {
      // Handle empty lines (preserve blank line spacing)
      if (paragraph.trim() === '') {
        lines.push('')
        continue
      }

      // Then word-wrap each paragraph to fit within maxWidth
      const words = paragraph.split(' ')
      let current = ''

      for (const word of words) {
        const test = current ? `${current} ${word}` : word
        const testWidth = font.widthOfTextAtSize(test, size)
        if (testWidth > maxWidth && current) {
          lines.push(current)
          current = word
        } else {
          current = test
        }
      }
      if (current) lines.push(current)
    }

    return lines
  }

  function textBox(
    label: string,
    content: string,
    xStart: number,
    yTop: number,
    boxWidth: number,
    fontSize = 9,
    lineHeight = 13,
    paddingTop = 28,
    paddingH = 10,
  ): number {
    const maxTextWidth = boxWidth - paddingH * 2
    const lines = wrapText(content, maxTextWidth, fontSize)

    // Minimum height: label row + at least 1 content line + bottom padding
    const contentHeight = lines.length * lineHeight
    const totalHeight = paddingTop + contentHeight + 10 // 10 = bottom padding

    const boxY = yTop - totalHeight

    box(xStart, boxY, boxWidth, totalHeight)
    text(label, xStart + paddingH, yTop - 12, 10, true)

    lines.forEach((line, i) => {
      text(line, xStart + paddingH, yTop - paddingTop - i * lineHeight, fontSize)
    })

    return boxY - 10 // return next y position
  }
  // ================= HEADER =================
  try {
    const logoBytes = await fetch(logoUrl).then((r) => r.arrayBuffer())
    const logo = await pdfDoc.embedPng(logoBytes)

    page.drawImage(logo, { x: 40, y: height - 70, height: 40, width: 20 })
  } catch {}

  text('INFINITY', 70, height - 45, 16, true)
  text('MEDICALS', 70, height - 60, 16, true)
  text(`Sheet ID: ${sheet.id || '-'}`, 70, height - 75)
  text('SERVICE REPORT', 320, height - 45, 13, true)
  text('Phone/Fax:(+965)2261 47 67', 320, height - 55, 9)
  text('Mob:(+965)50553092', 320, height - 65, 9)
  text('Email:kuwait@infinitymedicalkwt.com', 320, height - 75, 9)
  text('Website:www.infinitymedicalkwt.com', 320, height - 85, 9)

  y -= 70
  // ================= TIME =================
  const labourMin = sheet.labourTimeMin ?? 0
  const h = Math.floor(labourMin / 60)
  const m = labourMin % 60
  // ================= INFO =================
  box(40, y - 80, width - 80, 80)
  text(`Date: ${new Date(sheet.createdAt).toLocaleDateString()}`, 50, y - 20)
  text(`Service Engineer: ${sheet.engineer?.name ?? '-'}`, 50, y - 35)
  const serviceTypes = ['INSTALLATION', 'CONTRACT', 'WARRANTY', 'PAID SERVICE']
  const boxSize = 10
  const startX = 300
  let boxY = y - 10

  serviceTypes.forEach((type) => {
    // Draw checkbox
    page.drawRectangle({
      x: startX,
      y: boxY - boxSize + 2,
      width: boxSize,
      height: boxSize,
      borderWidth: 1,
      borderColor: rgb(0, 0, 0),
    })
    if (sheet.serviceType === 'PAID')
      if (type === 'PAID SERVICE') {
        page.drawText('X', {
          x: startX + 1,
          y: boxY - boxSize + 3,
          size: 12,
          font: bold,
          color: rgb(0, 0, 0),
        })
      }
    // If this is the selected service type, add a checkmark
    if (sheet.serviceType === type) {
      page.drawText('X', {
        x: startX + 1,
        y: boxY - boxSize + 3,
        size: 12,
        font: bold,
        color: rgb(0, 0, 0),
      })
    }

    // Draw the label
    page.drawText(type, {
      x: startX + boxSize + 4,
      y: boxY - boxSize + 1,
      size: 10,
      font,
      color: rgb(0, 0, 0),
    })

    // Move down for next checkbox
    boxY -= 16
  })
  text(`Labour Time: ${h}h ${m}m`, 50, y - 50)
  y -= 100

  // ================= CUSTOMER =================
  box(40, y - 90, width - 80, 90)
  text('Customer Information:', 50, y - 20, 10, true)
  text(`Customer: ${sheet.customer?.name ?? '-'}`, 50, y - 35)
  text(`Phone: ${sheet.customer?.phone ?? '-'}`, 50, y - 50)
  text(`Address: ${sheet.customer?.address || '-'}`, 50, y - 65)
  text('Machine Information:', 320, y - 20, 10, true)
  text(`Machine SN: ${sheet.machine?.serialNumber ?? '-'}`, 320, y - 35)
  text(`Model: ${sheet.machine?.model?.name ?? '-'}`, 320, y - 50)
  text(`Manufacturer: ${sheet.machine?.model?.manufacturer?.name ?? '-'}`, 320, y - 65)
  text(`Warranty: ${sheet.machine?.underWarranty ? 'YES' : 'NO'}`, 320, y - 80)

  y -= 100

  // ================= PROBLEM =================
  y = textBox('Problem Found / Comments', sheet.problemFound || '-', 40, y, width - 80)

  // ================= WORK =================
  y = textBox('Work Report', sheet.workReport || '-', 40, y, width - 80)

  // ================= SPARE TABLE =================
  text('Parts', 40, y, 10, true)
  y -= 5
  box(40, y - 20, width - 80, 20)

  const spX = [40, 320, 420]
  y = row(['Item', 'Qty', 'Price'], spX, y)
  let sum = 0
  if (sheet.spareParts?.length) {
    sheet.spareParts.forEach((sp) => {
      const price = Number(sp.price) || 0
      const discount = Number(sp.discounted) || 0
      const qty = Number(sp.quantity) || 0
      const finalPrice = Math.max(price - discount, 0)

      y = row(
        [
          sp.itemName ?? '-',
          String(qty),
          `${finalPrice.toFixed(2)} ${discount ? `(discount -${discount.toFixed(2)})` : ''}`,
        ],
        spX,
        y,
      )

      sum += finalPrice * qty
    })

    y = row(['Total Price', '', sum.toFixed(2)], spX, y)
  } else {
    y = row(['No spare parts', '', ''], spX, y)
  }

  y -= 25

  // ================= LASER TABLE =================
  text('LASER DATA', 40, y, 10, true)
  y -= 5
  box(40, y - 20, width - 80, 20)

  const lX = [40, 250, 420]
  y = row(['Type', 'Lamp', 'Voltage'], lX, y)

  if (sheet.laserData?.length) {
    sheet.laserData.forEach((l) => {
      y = row([l.laserType ?? '-', String(l.lampCounter), String(l.voltage)], lX, y)
    })
  } else {
    y = row(['No laser data', '', ''], lX, y)
  }

  y -= 35

  // ================= SIGNATURE =================
  box(40, y - 90, width - 80, 90)
  text(`Customer Name:${sheet.customer.name}`, 60, y - 15, 10, true)
  text('Customer Signature', 60, y - 30, 10, true)
  text(
    `Signed Date: ${new Date(sheet.completionTime || '01/01/1977').toLocaleDateString()}`,
    350,
    y - 15,
    10,
    true,
  )
  if (sheet.customerSignature) {
    const png = sheet.customerSignature.split(',')[1]
    if (png) {
      const imageBytes = Uint8Array.from(atob(png), (c) => c.charCodeAt(0))
      const image = await pdfDoc.embedPng(imageBytes)

      page.drawImage(image, {
        x: 60,
        y: safeY(y - 75),
        width: 180,
        height: 35,
      })
    }
  }

  // ================= SAVE PDF =================
  const pdfBytes = await pdfDoc.save()
  return pdfBytes // Uint8Array directly
}
async function shareJobSheet() {
  const sheet = jobSheetStore.jobSheetDetail
  if (!sheet) return

  // Generate PDF
  const pdfBytes = await generateServicePDF()
  if (!pdfBytes) return
  // Wrap in Uint8Array
  const uint8 = new Uint8Array(pdfBytes)

  // Create a Blob for PDF
  const blob = new Blob([uint8], { type: 'application/pdf' })
  const url = URL.createObjectURL(blob)

  // DEV: Open in a new tab
  if (import.meta.env.DEV) {
    window.open(url, '_blank')
    return
  }

  // Create proper PDF file
  const file = new File([uint8], `ServiceReport-${sheet.id}.pdf`, {
    type: 'application/pdf', // ensures it's recognized as PDF
  })

  // Check if sharing supports files
  if (navigator.canShare && navigator.canShare({ files: [file] })) {
    await navigator.share({
      title: `Service Report ${sheet.id} ${sheet.customer.name}`,
      text: 'Service report attached',
      files: [file],
    })
  } else {
    // Fallback: download PDF
    const blob = new Blob([file], { type: 'application/pdf' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `ServiceReport-${sheet.id}-${sheet.customer.name}.pdf`
    a.click()
    URL.revokeObjectURL(url)
    alert('Native share not supported. PDF downloaded instead.')
  }
  // Cleanup
  URL.revokeObjectURL(url)
}

// route
const route = useRoute()
const changed = ref(false)
// ---- time helpers ----
const calcLabourMinutes = () => {
  const arrival = jobSheetStore.jobSheetDetail?.arrivalTime
  const completion = jobSheetStore.jobSheetDetail?.completionTime

  if (!arrival || !completion) return null

  const a = new Date(arrival).getTime()
  const c = new Date(completion).getTime()

  if (Number.isNaN(a) || Number.isNaN(c)) return null
  if (c < a) return null

  return Math.round((c - a) / 60000) // minutes
}

// Auto-calc labour time whenever arrival/completion changes
watch(
  () => [jobSheetStore.jobSheetDetail?.arrivalTime, jobSheetStore.jobSheetDetail?.completionTime],
  () => {
    if (!jobSheetStore.jobSheetDetail) return

    const mins = calcLabourMinutes()
    if (mins === null) return

    jobSheetStore.jobSheetDetail.labourTimeMin = mins
  },
  { deep: true },
)

onMounted(() => {
  if (!route.params.id) {
    Swal.fire('Error', 'No sheet ID provided', 'error')
    return
  }
  loading.value = true
  const id = route.params.id.toString()
  jobSheetStore
    .fetchJobSheet(id)
    .then(() => {
      jobSheetStore.fetchAttachments(id)
    })
    .finally(() => {
      loading.value = false
      changed.value = false
    })
})

watch(
  () => jobSheetStore.jobSheetDetail,
  () => {
    changed.value = true
  },
  { deep: true },
)

const handleReset = () => {
  if (!route.params.id) {
    Swal.fire('Error', 'No sheet ID provided', 'error')
    return
  }
  loading.value = true
  jobSheetStore.fetchJobSheet(route.params.id.toString()).finally(() => {
    loading.value = false
    changed.value = false
  })
}

// ---- totals ----
const sparePartsTotal = computed(() => {
  const parts = jobSheetStore.jobSheetDetail?.spareParts || []

  return parts.reduce((sum: number, sp: any) => {
    const price = Number(sp.price) || 0
    const discount = Number(sp.discounted) || 0
    const qty = Number(sp.quantity) || 0

    const finalPrice = Math.max(price - discount, 0) // prevent negative
    return sum + finalPrice * qty
  }, 0)
})

// ---- add spare part ----
const handleAddSparePart = (payload: any) => {
  if (!jobSheetStore.jobSheetDetail) return

  if (!jobSheetStore.jobSheetDetail.spareParts) jobSheetStore.jobSheetDetail.spareParts = []

  jobSheetStore.jobSheetDetail.spareParts.push({
    itemName: payload.itemName,
    quantity: payload.quantity,
    price: payload.price,

    discounted: payload.discounted,
  })

  addSparePartOpen.value = false
}

// ---- delete spare part (local only) ----
const removeSparePart = (index: number) => {
  if (!jobSheetStore.jobSheetDetail?.spareParts) return
  jobSheetStore.jobSheetDetail.spareParts.splice(index, 1)
}

// ---- add laser data ----
const handleAddLaserData = (payload: any) => {
  if (!jobSheetStore.jobSheetDetail) return
  if (!jobSheetStore.jobSheetDetail.laserData) jobSheetStore.jobSheetDetail.laserData = []

  jobSheetStore.jobSheetDetail.laserData.push({
    laserType: payload.laserType,
    lampCounter: payload.lampCounter,
    voltage: payload.voltage,
  })

  addLaserDataOpen.value = false
}

const removeLaserData = (index: number) => {
  if (!jobSheetStore.jobSheetDetail?.laserData) return
  jobSheetStore.jobSheetDetail.laserData.splice(index, 1)
}

// ---- SAVE ----
const handleEdit = () => {
  if (!route.params.id) {
    Swal.fire('Error', 'No sheet ID provided', 'error')
    return
  }

  const payload = {
    checkInTime: jobSheetStore.jobSheetDetail?.checkInTime,
    arrivalTime: jobSheetStore.jobSheetDetail?.arrivalTime,
    completionTime: jobSheetStore.jobSheetDetail?.completionTime,
    labourTimeMin: jobSheetStore.jobSheetDetail?.labourTimeMin,
    problemFound: jobSheetStore.jobSheetDetail?.problemFound,
    workReport: jobSheetStore.jobSheetDetail?.workReport,
    total: jobSheetStore.jobSheetDetail?.total,
    totalAfterDisc: jobSheetStore.jobSheetDetail?.totalAfterDisc,
    customerSignature: jobSheetStore.jobSheetDetail?.customerSignature,
    serviceType: jobSheetStore.jobSheetDetail?.serviceType,

    spareParts:
      jobSheetStore.jobSheetDetail?.spareParts?.map((sp) => ({
        itemName: sp.itemName,
        quantity: sp.quantity,
        price: sp.price,
        discounted: sp.discounted,
      })) || [],

    laserData:
      jobSheetStore.jobSheetDetail?.laserData?.map((ld) => ({
        laserType: ld.laserType,
        lampCounter: ld.lampCounter,
        voltage: ld.voltage,
      })) || [],
  }

  const routeId = route.params.id
  if (!routeId) return

  loading.value = true
  jobSheetStore.updateJobSheet(routeId.toString(), { ...payload }).then(() => {
    jobSheetStore.fetchJobSheet(routeId.toString()).finally(() => {
      loading.value = false
      changed.value = false
    })
  })
}
</script>

<template>
  <section class="text-center md:text-left h-full space-y-6">
    <!-- Loading State -->
    <div
      v-if="loading"
      class="flex flex-col text-center items-center justify-center animate-pulse w-full space-y-4 h-full"
    >
      <div
        class="w-8 h-8 border-4 border-primary/40 border-t-transparent rounded-full animate-spin"
      ></div>
      <p class="text-sm text-center animate-pulse">Loading Sheet..</p>
    </div>

    <!-- jobSheets List -->
    <div
      v-else-if="jobSheetStore.jobSheetDetail"
      class="flex flex-col gap-3 grid md:grid-cols-2 gap3"
    >
      <!-- Sheet ID -->
      <div class="card h-auto md:col-span-2 space-y-2">
        <h2 class="header-small text-left">
          <span class="font-black text-sm md:text-base">{{
            jobSheetStore.jobSheetDetail.purchaseOrderNo ? 'Purchase Order Number:' : 'Sheet ID:'
          }}</span
          >{{ jobSheetStore.jobSheetDetail.purchaseOrderNo || jobSheetStore.jobSheetDetail.id }}
        </h2>
        <h3 v-if="jobSheetStore.jobSheetDetail.createdAt" class="text-teritiary text-sm">
          <span class="font-bold">Created:</span
          >{{
            new Date(jobSheetStore.jobSheetDetail.createdAt).toLocaleDateString('en', {
              day: '2-digit',
              month: 'short',
              year: '2-digit',
            })
          }}
        </h3>
        <h3 v-if="jobSheetStore.jobSheetDetail.updatedAt" class="text-teritiary text-sm">
          <span class="font-bold">Updated:</span
          >{{
            new Date(jobSheetStore.jobSheetDetail.updatedAt).toLocaleDateString('en', {
              day: '2-digit',
              month: 'short',
              year: '2-digit',
            })
          }}
        </h3>
        <div class="mb-4 flex gap-4 w-full">
          <div class="">
            <label>Service Type</label>
            <select
              v-model="jobSheetStore.jobSheetDetail.serviceType"
              class="infinity-text-input"
              required
            >
              <option value="" disabled>Select a Service Type</option>
              <option value="INSTALLATION">INSTALLATION</option>
              <option value="CONTRACT">CONTRACT</option>
              <option value="WARRANTY">WARRANTY</option>
              <option value="PAID">PAID</option>
            </select>
          </div>
        </div>
        <button
          v-if="jobSheetStore.jobSheetDetail.callId"
          class="btn-lg w-full"
          @click="
            $router.push({
              name: 'single-call',
              params: { id: jobSheetStore.jobSheetDetail.callId },
            })
          "
        >
          Visit Related Call
        </button>
        <button class="btn-lg-outline w-full" @click="shareJobSheet">
          Share Service Report PDF
        </button>
      </div>
      <!-- Supervising Engineer -->
      <div class="card h-auto space-y-2 items-center md:items-start">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Supervised by:</span>
        </h2>
        <h2 class="text-primary text-sm md:text-lg">
          {{ jobSheetStore.jobSheetDetail.engineer.name }}
        </h2>
        <p>{{ jobSheetStore.jobSheetDetail.engineer.email }}</p>
      </div>
      <!-- Customer  -->
      <div class="card h-auto space-y-2 items-center md:items-start">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">For Customer:</span>
        </h2>
        <h2 class="text-primary text-sm md:text-lg">
          {{ jobSheetStore.jobSheetDetail.customer.name }}
        </h2>
        <p>
          {{ jobSheetStore.jobSheetDetail.customer.address }} |
          {{ jobSheetStore.jobSheetDetail.customer.phone }}
        </p>
      </div>

      <!-- Time Data -->
      <div class="card h-auto space-y-3 items-center md:items-start">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Time Data:</span>
        </h2>

        <div class="grid md:grid-cols-3 w-full gap-4">
          <!-- Arrival -->
          <div class="space-y-2">
            <label class="block text-sm font-medium">Arrival Time</label>
            <VueDatePicker
              v-model="jobSheetStore.jobSheetDetail.arrivalTime"
              :enable-time-picker="true"
              :teleport="true"
              :clearable="true"
            />
          </div>

          <!-- Completion -->
          <div class="space-y-2">
            <label class="block text-sm font-medium">Completion Time</label>
            <VueDatePicker
              v-model="jobSheetStore.jobSheetDetail.completionTime"
              :enable-time-picker="true"
              :teleport="true"
              :clearable="true"
            />
          </div>
        </div>

        <!-- Labour -->
        <div class="flex flex-col md:flex-row w-full gap-3 md:items-end">
          <div class="flex-1">
            <label class="block text-sm font-medium">Labour Time (minutes)</label>
            <input
              type="number"
              class="infinity-text-input w-full"
              v-model.number="jobSheetStore.jobSheetDetail.labourTimeMin"
              disabled
            />
          </div>

          <p class="text-sm text-teritiary md:w-64">
            Labour time is automatically calculated from Arrival → Completion.
          </p>
        </div>
      </div>

      <!-- Machine  -->
      <div class="card h-auto space-y-2 items-center md:items-start">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Machine:</span>
        </h2>
        <h2 class="text-primary text-sm md:text-lg">
          Serial Number: {{ jobSheetStore.jobSheetDetail.machine.serialNumber }}
        </h2>
        <h2 class="text-primary text-sm md:text-lg">
          Model: {{ jobSheetStore.jobSheetDetail.machine.model?.name }}
        </h2>
        <h2 class="text-primary text-sm md:text-lg">
          Manufacturer: {{ jobSheetStore.jobSheetDetail.machine.model?.manufacturer?.name }}
        </h2>
        <p>
          {{
            jobSheetStore.jobSheetDetail.machine.underWarranty ? 'Under Warranty' : 'No Warranty'
          }}
        </p>
      </div>
      <!-- Reports  -->
      <div class="card h-auto md:row-span-2 items-center md:items-start">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Reports:</span>
        </h2>
        <div class="w-full">
          <label class="block text-sm font-medium">Problem Found</label>
          <textarea
            v-model="jobSheetStore.jobSheetDetail.problemFound"
            class="infinity-text-input w-full"
            placeholder="None Provided"
          ></textarea>
        </div>

        <div class="w-full">
          <label class="block text-sm font-medium">Work Report</label>
          <textarea
            v-model="jobSheetStore.jobSheetDetail.workReport"
            class="infinity-text-input w-full"
            placeholder="None Provided"
          ></textarea>
        </div>
      </div>

      <!-- Attachments -->
      <div class="card h-auto space-y-3 items-center md:items-start md:row-span-2">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Attachments:</span>
        </h2>

        <!-- Upload -->
        <input
          type="file"
          accept="image/*"
          class="infinity-text-input"
          @change="uploadFile"
          :disabled="uploading"
        />

        <p v-if="uploading" class="text-sm animate-pulse">Uploading...</p>

        <!-- List -->
        <div v-if="jobSheetStore.attachments.length" class="space-y-2 w-full">
          <div
            v-for="att in jobSheetStore.attachments"
            :key="att.id"
            class="flex items-center border rounded p-2 gap-3 w-full"
          >
            <!-- Inline preview for images -->
            <img v-if="att.fileType === 'image'" :src="att.url" class="h-12 rounded border" />
            <div class="flex justify-between items-center gap-2 w-full">
              <span class="font-bold capitalize">{{ att.fileType }} File</span>
              <a
                :href="att.url"
                target="_blank"
                class="btn-sm px-3 py-2 w-32 h-12 text-center flex items-center justify-center"
              >
                View
              </a>
            </div>
          </div>
        </div>

        <p v-else class="text-sm text-teritiary">No attachments uploaded</p>
      </div>
      <!-- Spare Parts -->
      <div class="card h-auto space-y-3 items-center md:items-start md:row-span-2 md:col-span-2">
        <div class="flex items-center justify-between w-full gap-3">
          <h2 class="text-primary text-sm md:text-lg">
            <span class="font-black text-sm md:text-base">Spare Parts:</span>
          </h2>

          <button class="btn-sm-outline w-32" @click="addSparePartOpen = true">Add</button>
        </div>

        <p class="text-sm text-teritiary w-full">
          Total: <span class="font-bold">{{ sparePartsTotal.toFixed(2) }}</span>
        </p>

        <div v-if="jobSheetStore.jobSheetDetail.spareParts?.length" class="space-y-2 w-full">
          <div
            v-for="(sp, idx) in jobSheetStore.jobSheetDetail.spareParts"
            :key="sp.id || idx"
            class="flex items-center justify-between border rounded p-2 gap-3 w-full"
          >
            <div class="flex flex-col">
              <p class="font-bold">{{ sp.itemName }}</p>
              <p class="text-sm text-teritiary">
                Qty: {{ sp.quantity }} | Unit: {{ (sp.price - (sp.discounted || 0)).toFixed(2) }}
                <span v-if="sp.discounted"> (Discount -{{ sp.discounted.toFixed(2) }})</span>
              </p>
            </div>

            <button class="btn-sm-outline w-24" @click="removeSparePart(idx)">Remove</button>
          </div>
        </div>

        <p v-else class="text-sm text-teritiary">No spare parts added</p>
      </div>

      <!-- Laser Data -->
      <div class="card h-auto space-y-3 items-center md:items-start md:col-span-2">
        <div class="flex items-center justify-between w-full gap-3">
          <h2 class="text-primary text-sm md:text-lg">
            <span class="font-black text-sm md:text-base">Laser Data:</span>
          </h2>

          <button class="btn-sm-outline w-32" @click="addLaserDataOpen = true">Add</button>
        </div>

        <div v-if="jobSheetStore.jobSheetDetail.laserData?.length" class="space-y-2 w-full">
          <div
            v-for="(ld, idx) in jobSheetStore.jobSheetDetail.laserData"
            :key="ld.id || idx"
            class="flex items-center justify-between border rounded p-2 gap-3 w-full"
          >
            <div class="flex flex-col">
              <p class="font-bold">{{ ld.laserType }}</p>
              <p class="text-sm text-teritiary">
                Lamp: {{ ld.lampCounter ?? '-' }} | Voltage: {{ ld.voltage ?? '-' }}
              </p>
            </div>

            <button class="btn-sm-outline w-24" @click="removeLaserData(idx)">Remove</button>
          </div>
        </div>

        <p v-else class="text-sm text-teritiary">No laser data added</p>
      </div>

      <!-- Signature -->
      <div class="card space-y-3 items-center md:items-start md:col-span-2">
        <h2 class="text-primary text-sm md:text-lg">
          <span class="font-black text-sm md:text-base">Customer Signature:</span>
        </h2>

        <div class="w-full space-y-3">
          <!-- wrapper forces stable height -->
          <div class="w-full h-40">
            <Vue3Signature
              ref="signaturePad"
              class="border rounded-lg w-full h-full bg-white overflow-hidden"
              :sigOption="{
                penColor: 'black',
                backgroundColor: 'white',
              }"
              style="height: 100%; width: 100%"
            />
          </div>

          <div class="flex flex-col md:flex-row gap-2 w-full">
            <button class="btn-lg-outline flex-1" @click="clearSignature">Clear</button>
            <button class="btn-lg flex-1" @click="saveSignature">Save Signature</button>
          </div>

          <!-- Preview -->
          <div v-if="jobSheetStore.jobSheetDetail?.customerSignature" class="space-y-2 w-full">
            <p class="text-sm text-teritiary">Saved Signature Preview:</p>
            <img
              :src="jobSheetStore.jobSheetDetail.customerSignature"
              class="border rounded-lg max-h-40"
            />
          </div>
        </div>
      </div>

      <!-- Updates  -->
      <div v-if="changed" class="card h-auto flex-row md:col-span-2 gap-3">
        <button @click="handleReset" class="btn-lg-outline flex-1">Reset Changes</button>
        <button @click="handleEdit" class="btn-lg flex-1">Save Changes</button>
      </div>
    </div>
    <div v-else class="h-full card items-center justify-center w-full">
      <router-link class="btn-lg px-3 w-32 py-3 text-center" :to="{ name: 'dashboard-home' }">
        Back to Home
      </router-link>
    </div>
    <!-- Modals -->
    <AddSparePartModal
      :visible="addSparePartOpen"
      :loading="false"
      @close="addSparePartOpen = false"
      @submit="handleAddSparePart"
    />

    <AddLaserDataModal
      :visible="addLaserDataOpen"
      :loading="false"
      @close="addLaserDataOpen = false"
      @submit="handleAddLaserData"
    />
  </section>
</template>
