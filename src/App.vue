<script setup>
import { computed, ref } from "vue";
import shotTrackLogo from "./assets/shottrack-logo.png";
import vaccinesCsv from "../vaccines.csv?raw";

const pages = [
  { id: "home", label: "儀表板" },
  { id: "search", label: "重點清單" },
  { id: "scan", label: "幼兒資料" },
  { id: "verify", label: "人工關懷" },
];

const statusStyles = {
  已完成: "done",
  即將到期: "soon",
  逾期未完成: "overdue",
  待家長回報: "pending",
  等待家長確認: "pending",
};

const activePage = ref("home");
const keyword = ref("");
const classFilter = ref("");
const statusFilter = ref("");
const selectedId = ref("ST-2026-003");
const editingChildId = ref("");
const vaccineFormChildId = ref("");
const childForm = ref(createEmptyChild());
const vaccineForm = ref(createEmptyVaccine());

const children = ref([
  {
    id: "ST-2026-001",
    name: "林小安",
    className: "小海星班",
    guardians: [
      {
        name: "林怡君",
        phone: "0912-345-678",
        lineId: "linmom_0814",
        isPrimary: true,
      },
    ],
    vaccine: "MMR 第一劑",
    dueDate: "2026-05-20",
    status: "即將到期",
    reminder: "今日 08:30 已自動推播",
    lastUpdate: "家長尚未回報接種資料",
    action: "系統將於到期前 3 天再次提醒",
  },
  {
    id: "ST-2026-002",
    name: "陳以樂",
    className: "小太陽班",
    guardians: [
      {
        name: "陳柏宏",
        phone: "0922-118-336",
        lineId: "chen爸爸",
        isPrimary: true,
      },
    ],
    vaccine: "五合一 第三劑",
    dueDate: "2026-05-04",
    status: "已完成",
    reminder: "不需提醒",
    lastUpdate: "2026-05-06 家長已上傳接種紀錄",
    action: "已歸檔",
  },
  {
    id: "ST-2026-003",
    name: "黃妮妮",
    className: "小海星班",
    guardians: [
      {
        name: "黃雅雯",
        phone: "0988-765-432",
        lineId: "yhw_nini",
        isPrimary: true,
      },
      {
        name: "黃志明",
        phone: "0977-320-110",
        lineId: "nini_dad",
        isPrimary: false,
      },
    ],
    vaccine: "日本腦炎 第一劑",
    dueDate: "2026-05-01",
    status: "逾期未完成",
    reminder: "已自動推播 3 次",
    lastUpdate: "超過 11 天未完成回報",
    action: "建議今日人工關懷",
  },
  {
    id: "ST-2026-004",
    name: "吳晴晴",
    className: "小月亮班",
    guardians: [
      {
        name: "吳佳穎",
        phone: "0966-221-908",
        lineId: "sunnywu",
        isPrimary: true,
      },
    ],
    vaccine: "水痘 第一劑",
    dueDate: "2026-05-15",
    status: "等待家長確認",
    reminder: "昨日 18:00 已自動推播",
    lastUpdate: "家長已讀提醒，尚未上傳照片",
    action: "等待家長確認",
  },
  {
    id: "ST-2026-005",
    name: "張恩恩",
    className: "小太陽班",
    guardians: [
      {
        name: "張哲瑋",
        phone: "0933-852-019",
        lineId: "zwchang0602",
        isPrimary: true,
      },
    ],
    vaccine: "A 型肝炎 第一劑",
    dueDate: "2026-06-02",
    status: "已完成",
    reminder: "不需提醒",
    lastUpdate: "2026-05-08 已同步健保紀錄",
    action: "已歸檔",
  },
]);

const vaccineOptions = parseVaccinesCsv(vaccinesCsv);

const filteredChildren = computed(() => {
  const text = keyword.value.trim().toLowerCase();

  return children.value.filter((child) => {
    const matchesKeyword =
      !text ||
      [
        child.id,
        child.name,
        child.className,
        primaryGuardian(child).name,
        primaryGuardian(child).phone,
        child.vaccine,
        child.status,
      ]
        .join(" ")
        .toLowerCase()
        .includes(text);

    const matchesClass =
      !classFilter.value || child.className === classFilter.value;
    const matchesStatus =
      !statusFilter.value || child.status === statusFilter.value;

    return matchesKeyword && matchesClass && matchesStatus;
  });
});

const classOptions = computed(() => [
  ...new Set(children.value.map((child) => child.className)),
]);
const statusOptions = computed(() => [
  ...new Set(children.value.map((child) => child.status)),
]);
const vaccineNameOptions = computed(() => [
  ...new Set(vaccineOptions.map((vaccine) => vaccine.uiName)),
]);
const selectedVaccineDoses = computed(() =>
  vaccineOptions.filter((vaccine) => vaccine.uiName === vaccineForm.value.name),
);

const stats = computed(() =>
  ["已完成", "即將到期", "逾期未完成", "等待家長確認"].map((status) => ({
    status,
    count: children.value.filter((child) =>
      status === "等待家長確認"
        ? ["等待家長確認", "待家長回報"].includes(child.status)
        : child.status === status,
    ).length,
  })),
);

const focusList = computed(() =>
  children.value.filter((child) => child.status !== "已完成"),
);

const autoReminderList = computed(() =>
  children.value.filter((child) =>
    ["即將到期", "待家長回報", "等待家長確認"].includes(child.status),
  ),
);

const manualCareList = computed(() =>
  children.value.filter((child) => child.status === "逾期未完成"),
);

const selectedChild = computed(() =>
  children.value.find((child) => child.id === selectedId.value.trim()),
);

function getStatusClass(status) {
  return statusStyles[status] ?? "pending";
}

function primaryGuardian(child) {
  return (
    child.guardians?.find((guardian) => guardian.isPrimary) ??
    child.guardians?.[0] ?? { name: "", phone: "", lineId: "" }
  );
}

function parseCsvLine(line) {
  const cells = [];
  let current = "";
  let insideQuotes = false;

  for (const char of line) {
    if (char === '"') {
      insideQuotes = !insideQuotes;
    } else if (char === "," && !insideQuotes) {
      cells.push(current.trim());
      current = "";
    } else {
      current += char;
    }
  }

  cells.push(current.trim());
  return cells;
}

function parseVaccinesCsv(csvText) {
  const [headerLine, ...rows] = csvText.trim().split(/\r?\n/);
  const headers = parseCsvLine(headerLine);
  const uiNameIndex = headers.indexOf("UI_Name");
  const doseIndex = headers.indexOf("Dose");
  const codeIndex = headers.indexOf("Sys_Code");

  return rows
    .map((row) => {
      const cells = parseCsvLine(row);

      return {
        uiName: cells[uiNameIndex],
        dose: cells[doseIndex],
        code: cells[codeIndex],
      };
    })
    .filter((vaccine) => vaccine.uiName && vaccine.dose);
}

function openCare(id) {
  selectedId.value = id;
  activePage.value = "verify";
}

function createEmptyChild() {
  return {
    id: "",
    name: "",
    className: "",
    guardians: [createEmptyGuardian(true)],
    vaccine: "",
    dueDate: "",
    status: "等待家長確認",
    reminder: "尚未發送提醒",
    lastUpdate: "尚未建立接種時程",
    action: "建立資料後由系統產生提醒",
  };
}

function createEmptyGuardian(isPrimary = false) {
  return {
    name: "",
    phone: "",
    lineId: "",
    isPrimary,
  };
}

function createEmptyVaccine() {
  return {
    name: "",
    dose: "",
    date: "",
    status: "等待家長確認",
  };
}

function generateChildId() {
  const nextNumber =
    Math.max(
      0,
      ...children.value.map(
        (child) => Number(child.id.replace("ST-2026-", "")) || 0,
      ),
    ) + 1;

  return `ST-2026-${String(nextNumber).padStart(3, "0")}`;
}

function startNewChild() {
  editingChildId.value = "";
  childForm.value = createEmptyChild();
  activePage.value = "scan";
}

function editChild(child) {
  editingChildId.value = child.id;
  childForm.value = {
    ...child,
    guardians: child.guardians?.length
      ? child.guardians.map((guardian) => ({ ...guardian }))
      : [createEmptyGuardian(true)],
  };
  activePage.value = "scan";
}

function saveChild() {
  if (!childForm.value.name || !childForm.value.className) return;

  const payload = {
    ...childForm.value,
    id: editingChildId.value || generateChildId(),
    guardians: normalizeGuardians(childForm.value.guardians),
    reminder:
      childForm.value.status === "已完成"
        ? "不需提醒"
        : childForm.value.reminder || "系統將依到期日自動推播",
    lastUpdate: childForm.value.lastUpdate || "托育端已建立幼兒資料",
    action: childForm.value.action || "系統自動追蹤接種時程",
  };

  const existingIndex = children.value.findIndex(
    (child) => child.id === editingChildId.value,
  );

  if (existingIndex >= 0) {
    children.value[existingIndex] = payload;
  } else {
    children.value.push(payload);
  }

  selectedId.value = payload.id;
  editChild(payload);
}

function normalizeGuardians(guardians) {
  const cleaned = guardians
    .filter((guardian) => guardian.name || guardian.phone || guardian.lineId)
    .map((guardian, index) => ({
      ...guardian,
      isPrimary: guardian.isPrimary || index === 0,
    }));

  if (!cleaned.length) {
    return [createEmptyGuardian(true)];
  }

  if (!cleaned.some((guardian) => guardian.isPrimary)) {
    cleaned[0].isPrimary = true;
  }

  return cleaned;
}

function addGuardian() {
  childForm.value.guardians.push(createEmptyGuardian(false));
}

function removeGuardian(index) {
  if (childForm.value.guardians.length === 1) return;

  const wasPrimary = childForm.value.guardians[index].isPrimary;
  childForm.value.guardians.splice(index, 1);

  if (wasPrimary) {
    childForm.value.guardians[0].isPrimary = true;
  }
}

function setPrimaryGuardian(index) {
  childForm.value.guardians = childForm.value.guardians.map(
    (guardian, guardianIndex) => ({
      ...guardian,
      isPrimary: guardianIndex === index,
    }),
  );
}

function startVaccineForm(child) {
  vaccineFormChildId.value = child.id;
  vaccineForm.value = createEmptyVaccine();
  activePage.value = "search";
}

function saveVaccine() {
  const child = children.value.find(
    (item) => item.id === vaccineFormChildId.value,
  );

  if (
    !child ||
    !vaccineForm.value.name ||
    !vaccineForm.value.dose ||
    !vaccineForm.value.date
  ) {
    return;
  }

  child.vaccine = `${vaccineForm.value.name} ${vaccineForm.value.dose}`;
  child.dueDate = vaccineForm.value.date;
  child.status = vaccineForm.value.status;
  child.reminder = "新增接種資訊後，系統等待家長確認";
  child.lastUpdate = `${vaccineForm.value.date} 已建立接種資訊`;
  child.action = "等待家長確認接種紀錄";
  child.vaccinations = [
    ...(child.vaccinations ?? []),
    {
      ...vaccineForm.value,
      createdAt: new Date().toISOString(),
    },
  ];

  vaccineForm.value = createEmptyVaccine();
}

function updateSelectedVaccineName() {
  vaccineForm.value.dose = "";
}

function markAsCompleted(child) {
  child.status = "已完成";
  child.reminder = "不需提醒";
  child.lastUpdate = "剛剛已由托育端確認家長回報";
  child.action = "已歸檔";
}

function markCareDone() {
  if (!selectedChild.value) return;

  selectedChild.value.status = "等待家長確認";
  selectedChild.value.reminder = "人工關懷後已重新發送推播";
  selectedChild.value.lastUpdate = "托育人員已完成電話關懷";
  selectedChild.value.action = "等待家長確認";
}
</script>

<template>
  <main class="app">
    <aside class="sidebar">
      <div class="brand">
        <img :src="shotTrackLogo" alt="ShotTrack logo" />
        <div>
          <strong>一針不漏ShotTrack</strong>
          <small>托育端疫苗接種追蹤</small>
        </div>
      </div>

      <nav aria-label="主要功能">
        <button
          v-for="page in pages"
          :key="page.id"
          :class="{ active: activePage === page.id }"
          type="button"
          @click="activePage = page.id"
        >
          {{ page.label }}
        </button>
      </nav>
    </aside>

    <section class="workspace">
      <header class="topbar">
        <div class="title-block">
          <p class="label">自動化、即時化、低負擔</p>
          <h1>一針不漏ShotTrack 托育端疫苗接種追蹤</h1>
        </div>
        <img class="topbar-logo" :src="shotTrackLogo" alt="" />
        <div class="operator">
          <span>今日需人工處理</span>
          <strong>{{ manualCareList.length }} 件</strong>
        </div>
      </header>

      <section v-if="activePage === 'home'" class="dashboard">
        <article class="panel overview">
          <div class="panel-title">
            <div>
              <p class="label">Auto Dashboard</p>
              <h2>自動化狀態儀表板</h2>
            </div>
            <span class="sync-pill">系統已自動更新時程</span>
          </div>

          <div class="stats">
            <div
              v-for="item in stats"
              :key="item.status"
              :class="getStatusClass(item.status)"
            >
              <span>{{ item.status }}</span>
              <strong>{{ item.count }}</strong>
            </div>
          </div>
        </article>

        <article class="panel focus-panel">
          <div class="panel-title">
            <div>
              <p class="label">Priority List</p>
              <h2>今日重點清單</h2>
            </div>
            <button type="button" @click="activePage = 'search'">
              查看全部
            </button>
          </div>

          <div class="task-list">
            <article
              v-for="child in focusList"
              :key="child.id"
              class="task-row"
            >
              <div>
                <span class="badge" :class="getStatusClass(child.status)">
                  {{ child.status }}
                </span>
                <h3>{{ child.name }}</h3>
                <p>
                  {{ child.className }} · {{ child.vaccine }} · 接種/到期時間
                  {{ child.dueDate }}
                </p>
              </div>
              <div class="record-buttons">
                <button
                  class="secondary"
                  type="button"
                  @click="startVaccineForm(child)"
                >
                  新增疫苗資訊
                </button>
                <button type="button" @click="openCare(child.id)">處理</button>
              </div>
            </article>
          </div>
        </article>

        <article class="panel reminder-panel">
          <div class="panel-title">
            <div>
              <p class="label">Auto Reminder</p>
              <h2>系統代發提醒</h2>
            </div>
          </div>

          <div class="reminder-summary">
            <strong>{{ autoReminderList.length }}</strong>
            <p>件由系統自動推播中，托育人員不用逐一通知。</p>
          </div>

          <div class="quiet-list">
            <p v-for="child in autoReminderList" :key="child.id">
              {{ child.name }} · {{ child.reminder }}
            </p>
          </div>
        </article>
      </section>

      <section v-else-if="activePage === 'search'" class="panel">
        <div class="panel-title">
          <div>
            <p class="label">List Management</p>
            <h2>低門檻清單式管理</h2>
          </div>
          <div class="toolbar-actions">
            <input
              v-model="keyword"
              type="search"
              placeholder="搜尋幼兒、家長或疫苗"
            />
            <select v-model="classFilter" aria-label="依班級篩選">
              <option value="">全部班級</option>
              <option
                v-for="className in classOptions"
                :key="className"
                :value="className"
              >
                {{ className }}
              </option>
            </select>
            <select v-model="statusFilter" aria-label="依疫苗狀態篩選">
              <option value="">全部疫苗狀態</option>
              <option
                v-for="status in statusOptions"
                :key="status"
                :value="status"
              >
                {{ status }}
              </option>
            </select>
            <button type="button" @click="startNewChild">新增小朋友</button>
          </div>
        </div>

        <form
          v-if="vaccineFormChildId"
          class="vaccine-form"
          @submit.prevent="saveVaccine"
        >
          <div>
            <p class="label">Vaccination</p>
            <h2>新增幼兒接種資訊</h2>
            <p>
              {{
                children.find((child) => child.id === vaccineFormChildId)?.name
              }}
              · 目前狀態預設等待家長確認
            </p>
          </div>

          <label>
            <span>接種名稱</span>
            <select
              v-model="vaccineForm.name"
              required
              @change="updateSelectedVaccineName"
            >
              <option value="" disabled>選擇接種名稱</option>
              <option
                v-for="name in vaccineNameOptions"
                :key="name"
                :value="name"
              >
                {{ name }}
              </option>
            </select>
          </label>
          <label>
            <span>接種劑次</span>
            <select
              v-model="vaccineForm.dose"
              required
              :disabled="!vaccineForm.name"
            >
              <option value="" disabled>選擇接種劑次</option>
              <option
                v-for="vaccine in selectedVaccineDoses"
                :key="vaccine.code"
                :value="`第 ${vaccine.dose} 劑`"
              >
                第 {{ vaccine.dose }} 劑
              </option>
            </select>
          </label>
          <label>
            <span>接種時間</span>
            <input v-model="vaccineForm.date" required type="date" />
          </label>
          <label>
            <span>目前狀態</span>
            <select v-model="vaccineForm.status" required>
              <option>等待家長確認</option>
            </select>
          </label>

          <div class="form-actions">
            <button type="submit">儲存接種資訊</button>
            <button
              class="secondary"
              type="button"
              @click="vaccineFormChildId = ''"
            >
              取消
            </button>
          </div>
        </form>

        <div class="table-list">
          <article
            v-for="child in filteredChildren"
            :key="child.id"
            class="record"
          >
            <div class="record-main">
              <span class="badge" :class="getStatusClass(child.status)">
                {{ child.status }}
              </span>
              <div>
                <h3>{{ child.name }}</h3>
                <p>
                  {{ child.className }} · {{ primaryGuardian(child).name }} ·
                  {{ primaryGuardian(child).phone }}
                </p>
              </div>
            </div>
            <div class="record-detail">
              <strong>{{ child.vaccine }}</strong>
              <span>接種/到期時間 {{ child.dueDate }}</span>
              <small>{{ child.action }}</small>
            </div>
            <div class="record-buttons">
              <button
                class="secondary"
                type="button"
                @click="startVaccineForm(child)"
              >
                新增疫苗資訊
              </button>
              <button class="secondary" type="button" @click="editChild(child)">
                修改
              </button>
              <button type="button" @click="openCare(child.id)">查看</button>
            </div>
          </article>
        </div>
      </section>

      <section v-else-if="activePage === 'scan'" class="child-management">
        <article class="panel">
          <div class="panel-title">
            <div>
              <p class="label">Child Profile</p>
              <h2>
                {{ editingChildId ? "修改小朋友資料" : "新增小朋友資料" }}
              </h2>
            </div>
            <button class="secondary" type="button" @click="startNewChild">
              清空表單
            </button>
          </div>

          <form class="child-form" @submit.prevent="saveChild">
            <div class="system-id full-width">
              <span>系統編號</span>
              <strong>{{ editingChildId || "儲存後自動生成" }}</strong>
            </div>
            <label>
              <span>幼兒姓名</span>
              <input
                v-model="childForm.name"
                required
                placeholder="輸入幼兒姓名"
              />
            </label>
            <label>
              <span>班級</span>
              <select v-model="childForm.className" required>
                <option value="" disabled>選擇班級</option>
                <option>小海星班</option>
                <option>小太陽班</option>
                <option>小月亮班</option>
                <option>小彩虹班</option>
              </select>
            </label>
            <label>
              <span>追蹤狀態</span>
              <select v-model="childForm.status" required>
                <option>已完成</option>
                <option>即將到期</option>
                <option>逾期未完成</option>
                <option>等待家長確認</option>
              </select>
            </label>
            <section class="guardian-section full-width">
              <div class="subsection-title">
                <div>
                  <p class="label">Guardian</p>
                  <h3>家長資料</h3>
                </div>
                <button class="secondary" type="button" @click="addGuardian">
                  新增家長
                </button>
              </div>

              <article
                v-for="(guardian, index) in childForm.guardians"
                :key="index"
                class="guardian-card"
              >
                <div class="guardian-card-title">
                  <strong>家長 {{ index + 1 }}</strong>
                  <label class="primary-toggle">
                    <input
                      type="radio"
                      name="primary-guardian"
                      :checked="guardian.isPrimary"
                      @change="setPrimaryGuardian(index)"
                    />
                    <span>主要照顧者</span>
                  </label>
                </div>
                <label>
                  <span>家長姓名</span>
                  <input
                    v-model="guardian.name"
                    required
                    placeholder="輸入家長姓名"
                  />
                </label>
                <label>
                  <span>家長電話</span>
                  <input
                    v-model="guardian.phone"
                    required
                    placeholder="例如 0912-345-678"
                  />
                </label>
                <label>
                  <span>家長 Line ID</span>
                  <input
                    v-model="guardian.lineId"
                    required
                    placeholder="例如 parent_line_id"
                  />
                </label>
                <button
                  v-if="childForm.guardians.length > 1"
                  class="secondary"
                  type="button"
                  @click="removeGuardian(index)"
                >
                  移除此家長
                </button>
              </article>
            </section>
            <label class="full-width">
              <span>家長綁定與提醒紀錄</span>
              <input
                v-model="childForm.reminder"
                placeholder="例如 今日 08:30 已自動推播"
              />
            </label>
            <label class="full-width">
              <span>備註</span>
              <input
                v-model="childForm.lastUpdate"
                placeholder="例如 家長已讀提醒，尚未上傳照片"
              />
            </label>

            <div class="form-actions">
              <button type="submit">
                {{ editingChildId ? "儲存修改" : "新增小朋友" }}
              </button>
            </div>
          </form>
        </article>

        <article class="panel">
          <div class="panel-title">
            <div>
              <p class="label">Linked Data</p>
              <h2>班級與家長綁定</h2>
            </div>
          </div>

          <div class="binding-preview">
            <span class="badge" :class="getStatusClass(childForm.status)">
              {{ childForm.status }}
            </span>
            <h3>{{ childForm.name || "尚未輸入姓名" }}</h3>
            <dl>
              <div>
                <dt>班級</dt>
                <dd>{{ childForm.className || "尚未綁定" }}</dd>
              </div>
              <div>
                <dt>家長</dt>
                <dd>{{ primaryGuardian(childForm).name || "尚未綁定" }}</dd>
              </div>
              <div>
                <dt>電話</dt>
                <dd>{{ primaryGuardian(childForm).phone || "尚未填寫" }}</dd>
              </div>
              <div>
                <dt>Line ID</dt>
                <dd>{{ primaryGuardian(childForm).lineId || "尚未填寫" }}</dd>
              </div>
              <div>
                <dt>系統提醒</dt>
                <dd>{{ childForm.reminder || "儲存後依狀態產生提醒" }}</dd>
              </div>
            </dl>
          </div>
        </article>
      </section>

      <section v-else class="panel care-panel">
        <div class="panel-title">
          <div>
            <p class="label">Manual Care</p>
            <h2>特殊個案人工關懷</h2>
          </div>
          <select v-model="selectedId">
            <option v-for="child in children" :key="child.id" :value="child.id">
              {{ child.name }} / {{ child.status }}
            </option>
          </select>
        </div>

        <div v-if="selectedChild" class="care-card">
          <span class="badge" :class="getStatusClass(selectedChild.status)">
            {{ selectedChild.status }}
          </span>
          <h3>{{ selectedChild.name }}</h3>

          <dl>
            <div>
              <dt>班級</dt>
              <dd>{{ selectedChild.className }}</dd>
            </div>
            <div>
              <dt>家長</dt>
              <dd>
                {{ primaryGuardian(selectedChild).name }} ·
                {{ primaryGuardian(selectedChild).phone }} · Line
                {{ primaryGuardian(selectedChild).lineId }}
              </dd>
            </div>
            <div>
              <dt>接種項目</dt>
              <dd>{{ selectedChild.vaccine }}</dd>
            </div>
            <div>
              <dt>接種/到期時間</dt>
              <dd>{{ selectedChild.dueDate }}</dd>
            </div>
            <div>
              <dt>系統提醒</dt>
              <dd>{{ selectedChild.reminder }}</dd>
            </div>
            <div>
              <dt>目前狀態</dt>
              <dd>{{ selectedChild.lastUpdate }}</dd>
            </div>
          </dl>

          <div class="care-actions">
            <button type="button" @click="markCareDone">
              已人工關懷並重發提醒
            </button>
            <button
              class="secondary"
              type="button"
              @click="markAsCompleted(selectedChild)"
            >
              家長已回報，直接歸檔
            </button>
          </div>
        </div>
      </section>
    </section>
  </main>
</template>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(body) {
  margin: 0;
  background:
    radial-gradient(
      circle at 18% 8%,
      rgba(174, 217, 240, 0.55),
      transparent 34%
    ),
    linear-gradient(135deg, #eef8fd 0%, #f9fcfb 52%, #fff3f5 100%);
  color: #213547;
  font-family:
    Inter,
    ui-sans-serif,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    sans-serif;
}

button,
input,
select {
  font: inherit;
}

button {
  min-height: 40px;
  border: 0;
  border-radius: 8px;
  background: #2f88bd;
  color: #fff;
  cursor: pointer;
  font-weight: 800;
  padding: 0 14px;
  white-space: nowrap;
  box-shadow: 0 8px 18px rgba(47, 136, 189, 0.18);
}

input,
select {
  width: min(100%, 380px);
  min-height: 44px;
  border: 1px solid #c6dceb;
  border-radius: 8px;
  background: #fff;
  color: #213547;
  padding: 0 14px;
}

input:focus,
select:focus {
  border-color: #3f9ed2;
  box-shadow: 0 0 0 4px rgba(63, 158, 210, 0.14);
  outline: none;
}

.app {
  display: grid;
  min-height: 100vh;
  grid-template-columns: 248px 1fr;
}

.sidebar {
  display: flex;
  flex-direction: column;
  gap: 28px;
  padding: 24px 18px;
  border-right: 1px solid rgba(70, 150, 191, 0.18);
  background:
    linear-gradient(
      180deg,
      rgba(255, 255, 255, 0.92),
      rgba(221, 241, 249, 0.86)
    ),
    #f5fbfe;
  color: #1f5f8f;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand img {
  width: 58px;
  height: 58px;
  border-radius: 50%;
  box-shadow: 0 10px 24px rgba(70, 150, 191, 0.22);
}

.brand div {
  display: grid;
  gap: 2px;
}

.brand small {
  color: #6b8fa4;
}

nav {
  display: grid;
  gap: 8px;
}

nav button {
  background: transparent;
  box-shadow: none;
  color: #517991;
  text-align: left;
}

nav button.active,
nav button:hover {
  background: #e7f5fc;
  color: #1f6fa6;
}

.workspace {
  padding: 28px;
}

.topbar,
.panel-title,
.record,
.record-main,
.care-actions {
  display: flex;
  gap: 16px;
}

.topbar,
.panel-title,
.record {
  justify-content: space-between;
  align-items: center;
}

.topbar {
  margin-bottom: 24px;
  border: 1px solid rgba(70, 150, 191, 0.18);
  border-radius: 8px;
  background:
    linear-gradient(
      100deg,
      rgba(255, 255, 255, 0.92),
      rgba(232, 247, 253, 0.86)
    ),
    #fff;
  box-shadow: 0 12px 32px rgba(70, 150, 191, 0.12);
  padding: 18px;
}

h1,
h2,
h3,
p {
  margin: 0;
}

h1 {
  margin-top: 4px;
  color: #1f5f8f;
  font-size: clamp(28px, 4vw, 42px);
  line-height: 1.12;
}

h2 {
  font-size: 23px;
}

h3 {
  font-size: 18px;
}

.label {
  color: #3f8fc0;
  font-size: 13px;
  font-weight: 800;
  letter-spacing: 0;
  text-transform: uppercase;
}

.muted,
.record p,
.record-detail span,
.record-detail small,
dt,
.quiet-list,
.task-row p {
  color: #667d8c;
}

.topbar-logo {
  width: 82px;
  height: 82px;
  flex: 0 0 auto;
  border-radius: 50%;
  box-shadow: 0 12px 26px rgba(70, 150, 191, 0.2);
}

.operator {
  display: grid;
  min-width: 142px;
  gap: 4px;
  border-left: 1px solid #cbdfea;
  padding-left: 18px;
}

.operator span {
  color: #667d8c;
  font-size: 13px;
}

.dashboard,
.child-management {
  display: grid;
  gap: 18px;
}

.dashboard {
  grid-template-columns: minmax(0, 1fr) minmax(340px, 0.62fr);
}

.overview,
.focus-panel {
  grid-column: span 1;
}

.focus-panel {
  grid-row: span 2;
}

.reminder-panel {
  grid-column: 1;
}

.child-management {
  grid-template-columns: minmax(320px, 0.8fr) minmax(0, 1fr);
}

.panel {
  border: 1px solid rgba(70, 150, 191, 0.18);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.92);
  box-shadow: 0 10px 24px rgba(70, 150, 191, 0.1);
  padding: 22px;
}

.stats {
  display: grid;
  grid-template-columns: repeat(4, minmax(120px, 1fr));
  gap: 12px;
  margin-top: 22px;
}

.stats div {
  display: grid;
  gap: 10px;
  min-height: 106px;
  border-radius: 8px;
  padding: 18px;
}

.stats span {
  color: #5f7888;
  font-size: 14px;
}

.stats strong {
  font-size: 34px;
}

.done {
  background: #e7f5ed;
  color: #2f7455;
}

.soon {
  background: #e6f4fb;
  color: #2b79ad;
}

.overdue {
  background: #ffe5eb;
  color: #b84963;
}

.pending {
  background: #eff4ff;
  color: #4b6ca8;
}

.sync-pill,
.badge {
  display: inline-flex;
  width: fit-content;
  align-items: center;
  min-height: 30px;
  border-radius: 999px;
  padding: 0 10px;
  font-size: 13px;
  font-weight: 800;
}

.sync-pill {
  background: #e6f4fb;
  color: #2b79ad;
}

.task-list,
.table-list,
.care-card,
.binding-preview,
.quiet-list {
  display: grid;
  gap: 12px;
}

.task-row {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: center;
  border: 1px solid #dcebf3;
  border-radius: 8px;
  padding: 14px;
}

.task-row div {
  display: grid;
  gap: 8px;
}

.reminder-summary {
  display: flex;
  align-items: baseline;
  gap: 10px;
  margin: 16px 0;
  border-radius: 8px;
  background: linear-gradient(90deg, #e6f4fb, #eef8f2);
  padding: 18px;
}

.reminder-summary strong {
  color: #2b79ad;
  font-size: 40px;
}

.quiet-list p {
  border-bottom: 1px solid #e2eef5;
  padding-bottom: 10px;
}

.record {
  border: 1px solid #dcebf3;
  border-radius: 8px;
  padding: 14px;
}

.record-main {
  align-items: center;
  min-width: 260px;
}

.record-detail {
  display: grid;
  gap: 4px;
  min-width: 240px;
}

.toolbar-actions,
.record-buttons,
.form-actions {
  display: flex;
  gap: 10px;
  align-items: center;
}

.toolbar-actions {
  flex-wrap: wrap;
  justify-content: flex-end;
}

.record-buttons {
  flex-shrink: 0;
}

.child-form {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
  margin-top: 22px;
}

.child-form label,
.vaccine-form label {
  display: grid;
  gap: 8px;
}

.child-form label span,
.vaccine-form label span {
  color: #5f7888;
  font-size: 14px;
  font-weight: 800;
}

.child-form input,
.child-form select,
.vaccine-form input,
.vaccine-form select {
  width: 100%;
}

.full-width,
.form-actions {
  grid-column: 1 / -1;
}

.form-actions {
  justify-content: flex-start;
}

.binding-preview {
  margin-top: 16px;
}

.system-id {
  display: flex;
  justify-content: space-between;
  gap: 14px;
  align-items: center;
  border: 1px solid #dcebf3;
  border-radius: 8px;
  background: #f4fbfe;
  padding: 12px 14px;
}

.system-id span {
  color: #5f7888;
  font-size: 14px;
  font-weight: 800;
}

.system-id strong {
  color: #2b79ad;
}

.guardian-section {
  display: grid;
  gap: 12px;
  border: 1px solid #dcebf3;
  border-radius: 8px;
  background: #fbfdfe;
  padding: 14px;
}

.subsection-title,
.guardian-card-title {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: center;
}

.guardian-card {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr)) auto;
  gap: 12px;
  border: 1px solid #e2eef5;
  border-radius: 8px;
  background: #fff;
  padding: 14px;
}

.guardian-card-title {
  grid-column: 1 / -1;
}

.primary-toggle {
  display: inline-flex;
  gap: 8px;
  align-items: center;
  color: #2b79ad;
  font-weight: 800;
}

.primary-toggle input {
  width: auto;
  min-height: auto;
}

.vaccine-form {
  display: grid;
  grid-template-columns: repeat(4, minmax(150px, 1fr));
  gap: 14px;
  margin: 0 0 18px;
  border: 1px solid #dcebf3;
  border-radius: 8px;
  background: linear-gradient(100deg, #f4fbfe, #fff7f9);
  padding: 16px;
}

.vaccine-form > div:first-child {
  grid-column: 1 / -1;
}

.empty {
  margin-top: 18px;
  border-radius: 8px;
  background: #e6f4fb;
  color: #2b79ad;
  padding: 16px;
}

dl {
  display: grid;
  gap: 10px;
  margin: 18px 0 0;
}

dl div {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  border-bottom: 1px solid #e2eef5;
  padding-bottom: 10px;
}

dd {
  margin: 0;
  font-weight: 800;
  text-align: right;
}

.care-actions {
  flex-wrap: wrap;
  margin-top: 18px;
}

.secondary {
  background: #edf7fb;
  box-shadow: none;
  color: #2b79ad;
}

@media (max-width: 960px) {
  .app {
    grid-template-columns: 1fr;
  }

  .sidebar {
    position: sticky;
    top: 0;
    z-index: 1;
    gap: 16px;
    padding: 18px;
  }

  nav {
    grid-template-columns: repeat(4, 1fr);
  }

  nav button {
    text-align: center;
  }

  .workspace {
    padding: 20px;
  }

  .dashboard,
  .child-management {
    grid-template-columns: 1fr;
  }

  .focus-panel,
  .reminder-panel {
    grid-column: auto;
  }

  .stats {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .topbar,
  .panel-title,
  .record,
  .task-row,
  .record-main,
  .toolbar-actions,
  .record-buttons,
  .subsection-title,
  .guardian-card-title {
    align-items: stretch;
    flex-direction: column;
  }

  .toolbar-actions input {
    width: 100%;
  }
}

@media (max-width: 560px) {
  nav,
  .stats {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .child-form {
    grid-template-columns: 1fr;
  }

  .guardian-card,
  .vaccine-form {
    grid-template-columns: 1fr;
  }

  dl div {
    display: grid;
  }

  dd {
    text-align: left;
  }
}
</style>
