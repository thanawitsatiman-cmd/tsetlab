/**
 * ระบบติดตามคำร้อง — ปรับให้รองรับข้อมูลหลายหมื่นแถว และเจ้าหน้าที่ 3–5 คนใช้พร้อมกัน
 *
 * แนวคิดหลัก
 *  1) อ่านชีตเฉพาะคอลัมน์ที่ใช้แสดงรายการ 
 *  2) เก็บข้อมูลใน CacheService เป็นก้อนเล็ก ๆ ตามลำดับแถวในชีต
 *  3) เพิ่ม / แก้ไข ใช้วิธี "แก้เฉพาะก้อนที่เกี่ยวข้องใน Cache" จึงไม่ต้องอ่านชีตใหม่ทั้งหมด
 *     (ลบแถว หรือแก้ในชีตโดยตรง จะล้าง Cache แล้วอ่านใหม่ครั้งเดียว)
 *  4) เขียนข้อมูลทุกครั้งอยู่ใน LockService กันเจ้าหน้าที่หลายคนเขียนชนกัน
 *  5) สรุปภาพรวม (Dashboard) ถูกเก็บใน Cache แยก จึงเปิดซ้ำได้เร็ว
 */

const CONFIG = {
  SPREADSHEET_ID: '1ZgugO-o6NgAIrrrXnFiqnKjlvPT16zoL3tijsP2ZVgM',
  SHEET_NAMES: ['คำร้อง2569', 'คำร้อง2568'],   // ชีตข้อมูลของแต่ละปี: ชื่อแรก = ชีตหลัก (รับรายการใหม่ปีปัจจุบัน) ปีใหม่ให้เพิ่มชื่อต่อท้าย
  TIMEZONE: 'Asia/Bangkok',
  CACHE_SECONDS: 300,          // อายุ Cache (วินาที)
  CHUNK_ROWS: [100, 40, 15],   // จำนวนแถวต่อก้อน ลองจากใหญ่ไปเล็กถ้าก้อนใหญ่เกินขนาด
  MAX_CHUNK_CHARS: 30000,      // ขนาดสูงสุดต่อก้อน (CacheService จำกัด 100KB ต่อค่า)
  DEFAULT_PAGE_SIZE: 50,
  MAX_PAGE_SIZE: 200,
  LOCK_WAIT_MS: 20000,
  USERS_SHEET: 'Users',        // ชีตรายชื่อเจ้าหน้าที่และสิทธิ์
  OPTIONS_SHEET: 'ตัวเลือก',    // (ไม่บังคับ) แท็บเก็บรายการ type / reason ให้เว็บอ่าน
  AUDIT_SHEET: 'AuditLog',     // ชีตบันทึกว่าใครทำอะไร
  AUTO_REGISTER: true,         // true = ใครล็อกอิน Google แล้วเปิดลิงก์ จะถูกเพิ่มในชีต Users ให้อัตโนมัติ
  AUTO_ROLE: 'editor',         // สิทธิ์ของคนที่ถูกเพิ่มอัตโนมัติ (viewer / editor)
  AUTO_DOMAIN: '',             // ถ้าใส่ เช่น 'gmail.com' จะเพิ่มอัตโนมัติเฉพาะอีเมลโดเมนนี้ (ว่าง = ทุกโดเมน)
  USER_CACHE_SECONDS: 300      // อายุแคชสิทธิ์ผู้ใช้ (แก้ชีต Users แล้วอาจรอสูงสุดเท่านี้)
};

CONFIG.SHEET_NAME = CONFIG.SHEET_NAMES[0];   // ชีตหลัก
const RN_STRIDE = 1000000;   // เลขแถวที่ส่งให้หน้าเว็บ = ลำดับชีต * RN_STRIDE + เลขแถว (ชีตหลักจึงเท่ากับเลขแถวจริง)
function encRn_(si, rowNo) { return (si || 0) * RN_STRIDE + rowNo; }

const REASON_LIST = [
  'รอตรวจ',
  'สายเมนไม่เรียยร้อย',
  'ยังไม่ได้เดินระบบ 3P',
  'รอเดินสายเมน',
  'รอโทรเเจ้งให้ชำระเงิน',
  'ทำ wom เสร็จแล้ว',
  'รอทำเอกสาร',
  'ตรวจผ่านเเล้วรอใบเเจ้งหนี้',
  'รอลูกค้าเซ็นเอกสาร',
  'ยังไม่ขยายเขต',
  'ชำระเงิน',
  'เอกสารไม่ครบ',
  'รอติดตั้ง',
  'รอเซ็น',
  'ทำเอกสารเพิ่มขนาด',
  'เเจ้งลูกค้า',
  'ยังไม่ได้เดินสายเมนเข้าเสาไฟฟ้า',
  'ให้เปลี่ยนสายเมนใหม่',
  'ยังไม่ได้เดินสายเมน',
  'สายเมนสั้นเกินไป',
  'ทำเรื่องขยายเขตปรับปรุงหม้อแปลง',
  'รอเดินสายเมนเข้าเสาไฟต้นใหม่ที่ขยายให้เรีบยร้อย',
  'ระบบไฟฟ้าภายในไม่เรียบร้อย',
  'เสาไฟฟ้าไม่ตรงหน้าบ้าน ต้องทำเรื่องปักเสาเเซม',
  'ให้เข้ามาทำเรื่องขยายเนื่องจากเสาไฟไม่ได้อยู่หน้าที่',
  'ใช้สายเมนไม่ถูกต้องตามมาตรฐาน,ปลายสายสั้นเกินไป',
  'ขนาดสายเมนไม่ถูกต้องตามมาตรฐาน ต้องเปลี่ยนเป็นเบอร์ 25',
  'รอขยายเขต',
  'ภายในยังไม่เรียบร้อย',
  'สายเมนไม่ถูกต้องมาตรฐาน',
  'ค้างเดินวงจรที่ 2',
  'สายเมนไม่ถูกต้อง',
  'ไม่ได้มัดสายเมนเข้าเเร็ค',
  'ให้เดินสายเมนใหม่',
  'ต้องยื่นขยายเขตเพิ่มเฟส',
  'รอตั้งตู้เบรกเกอร์',
  'เดินสายเมนไม่ถูกต้อง',
  'ต้องเดินสายเมนเข้าเสาไฟฟ้า',
  'รอแก้ไขสายเมนใหม่',
  'รอแก้ไขวงจรที่ 2',
  'ระบบไฟภายในยังไม่เสร็จ',
  'รอดึงสาย',
  'รอปรับปรุงระบบ',
  'รอติดตั้งวอลชาร์จ',
  'รอทำระบบ 3 เฟส,ดึงสาย',
  'รอทำเอกสาร ค้างค่าไฟ !',
  'ส่งเรื่องให้แผนกมิเตอร์เเล้ว',
  'ขาดเอกสารมอบอำนาจ',
  'สำรวจหน้างานก่อน',
  'ผชฟ เเจ้งว่าใส่สายมิเตอร์สลับ',
  'โทรเเจ้งล่วงหน้าก่อน',
  'ขาดสัญญาเช่า',
  'ไม่ได้มาร์เฟสที่ปลายสาย',
  'ขอด่วนมีงานขึ้นบ้านใหม่',
  'มีเมนเดิม',
  'ให้ใส่หัวงูเห่าและมาร์คปลายสาย',
  'ให้ใส่หัวงูเห่าและมาร์คปลายสาย',
  'รอเซ็น',
  'รอตารางโหลด',
  'แก้ไขเเล้ว',
  'ขาดสเปควอร์ชาจ /ใบจองรถ',
  'ผชฟ.เซ็น TOU',
  'เบอร์ไม่ถูกต้อง - ไม่สามารถติดต่อได้',
  'เขตป่าสงวน',
  'รอหัวหน้าตัดสินใจ',
  'ขาดเอกสาร Wallcharge',
  'รอติดแร็ค',
  'ขาดสำเนาทะเบียนบ้าน',
  'รอ ผชฟ. ติดสินใจ',
  'ไม่มีระบบแรงต่ำ',
  'เสนอผจก.',
  'รอถ่ายรูปตู้(เข้าดูหน้างานไม่ได้)',
  'ยังไม่ได้รวบสายเมน',
  'ขาดอากรสแตมป์',
  'มีมิเตอร์อยู่เเล้ว',
  'ต้องทำเรื่องปักเสาแซมไลน์',
  'นัดตรวจ',
  'เสาเอนเอียง สายหย่อน',
  'ไม่มีระบบ 3 เฟส',
  'ทำเอกสารเพิ่มขนาดเเล้ว'
];

/** ประเภทคำร้อง (เลือกได้ 1 ตัวเลือกต่อคำร้อง) — Apps Script อ่าน dropdown แบบคอลัมน์ชนิดตารางของชีตไม่ได้ จึงเก็บรายการไว้ที่นี่
 *  ถ้ามีแท็บ 'ตัวเลือก' ที่มีคอลัมน์ type จะใช้รายการจากแท็บนั้นแทน */
const TYPE_LIST = [
  'EV Charger',
  'ขอติดตั้งใหม่',
  'ขอใช้ไฟชั่วคราว',
  'เพิ่มขนาด',
  'เลิกใช้',
  'ย้าย',
  'เพิ่ม+ย้าย',
  'โอน+ย้าย',
  'เปลี่ยนประเภท',
  'โอน+ย้าย+เพิ่ม',
  'โอน+เปลี่ยนประเภท TOU',
  'โอน+เพิ่มEV',
  'ลดขนาด',
  'TOU',
  'เพิ่มขนาด EV',
  'รับเงินประกัน',
  'ย้าย+เพิ่ม EV',
  'ตั้งใหม่ + เปลี่ยนประเภท',
  'TOU + EV',
  'เพิ่มขนาด + EV + TOU',
  'โอน + เพิ่ม + EV + TOU',
  'ขอเปลี่ยนชั่วคราวเป็นอัตราประเภท 1',
  'เปลี่ยนอัตราที่ป่า',
  'ประกอบ มป. 160KVA'
];

const COMMA_OPTS_ = REASON_LIST.filter(x => x.indexOf(',') >= 0).sort((a, b) => b.length - a.length);

const HEADERS = [
  'ID', 'date', 'no', 'name', 'address', 'phone', 'type', 'status', 'phase',
  'size', 'inPhase', 'pole', 'route', 'note', 'src', 'reason'
];
const LIST_COLS = 15; // ID … src ส่วน reason (สาเหตุ) เป็นคอลัมน์ที่ 17 เก็บแยกในแถวย่อ
const COL = {};
HEADERS.forEach((h, i) => { COL[h] = i; });

const FIELD_LABELS = {
  date: 'วันที่', no: 'เลขที่', name: 'ชื่อ', address: 'ที่อยู่', phone: 'เบอร์โทร',
  type: 'ประเภท', phase: 'เฟส', size: 'ขนาดมิเตอร์',
  inPhase: 'เฟสเชื่อมต่อ', pole: 'ขนาดเสา', route: 'สายจดหน่วย', note: 'หมายเหตุ', reason: 'สาเหตุ'
};

const K_META = 'rq_meta2';   // รายละเอียดของ Cache ชุดปัจจุบัน
const K_GEN = 'rq_gen';     // เลขรุ่น เปลี่ยนทุกครั้งที่มีการเขียนข้อมูล
const K_HDR = 'rq_hdr_ok4'; // ตรวจหัวตารางแล้ว
const K_OPTS = 'rq_opts';   // รายการตัวเลือกประเภท/สาเหตุที่อ่านจาก dropdown ในชีต

/* ============================================================
 * เว็บแอป
 * ============================================================ */

function doGet() {
  return HtmlService.createHtmlOutputFromFile('index')
    .setTitle('ระบบติดตามคำร้อง')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

/* ============================================================
 * ผู้ใช้และสิทธิ์ (ตรวจฝั่งเซิร์ฟเวอร์ ทุกฟังก์ชันที่หน้าเว็บเรียกได้ต้องผ่านด่านนี้)
 *  viewer = ดูอย่างเดียว | editor = ดู/เพิ่ม/แก้ | admin = ทำได้ทุกอย่างรวมลบ
 * ============================================================ */

const ROLE_LEVEL = { viewer: 1, editor: 2, admin: 3 };
const USERS_HEADERS = ['email', 'name', 'role', 'active'];
const AUDIT_HEADERS = ['time', 'email', 'action', 'target', 'detail'];

/** ค่าในคอลัมน์ active: เว้นว่าง/TRUE/ติ๊กถูก = ใช้งานได้ ส่วน FALSE/ไม่/no/0/ปิด = ระงับ */
function parseActive_(v) {
  if (v === true) return true;
  if (v === false) return false;
  const t = String(v == null ? '' : v).trim().toLowerCase();
  if (t === '') return true;
  return ['false', 'no', 'n', '0', 'ไม่', 'ไม่ใช้งาน', 'ปิด', 'inactive'].indexOf(t) < 0;
}

/** ค่าในคอลัมน์ role: รับทั้งอังกฤษ/ไทย เว้นว่าง = editor คืน '' ถ้าไม่รู้จัก */
function parseRole_(v) {
  const t = String(v == null ? '' : v).trim().toLowerCase();
  if (t === '') return 'editor';
  if (ROLE_LEVEL[t]) return t;
  if (/แอดมิน|ผู้ดูแล/.test(t)) return 'admin';
  if (/แก้ไข|เจ้าหน้าที่|พนักงาน/.test(t)) return 'editor';
  if (/ดู/.test(t)) return 'viewer';
  return '';
}

/** หาผู้ใช้จากอีเมลที่ล็อกอิน: คืน { user, email, reason } (reason: noemail | nosheet | notfound | inactive | badrole) */
function resolveUser_() {
  const email = (Session.getActiveUser().getEmail() || '').toLowerCase().trim();
  if (!email) return { user: null, email: '', reason: 'noemail' };

  const cache = CacheService.getScriptCache();
  const key = 'user:' + email;
  const hit = cache.get(key);
  if (hit) { try { return { user: JSON.parse(hit), email: email, reason: '' }; } catch (e) { /* อ่านชีตใหม่ */ } }

  const sh = SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID).getSheetByName(CONFIG.USERS_SHEET);
  if (!sh || sh.getLastRow() < 2) return { user: null, email: email, reason: 'nosheet' };
  const rows = sh.getRange(2, 1, sh.getLastRow() - 1, 4).getValues();
  let reason = 'notfound';
  for (let i = 0; i < rows.length; i++) {
    const e = String(rows[i][0]).toLowerCase().trim();
    if (e !== email) continue;
    if (!parseActive_(rows[i][3])) { reason = 'inactive'; continue; }
    const role = parseRole_(rows[i][2]);
    if (!role) { reason = 'badrole'; continue; }
    const user = { email: email, name: String(rows[i][1] || email), role: role };
    cache.put(key, JSON.stringify(user), CONFIG.USER_CACHE_SECONDS);
    return { user: user, email: email, reason: '' };
  }
  if (reason === 'notfound' && autoRegister_(sh, email)) {
    const user = { email: email, name: email.split('@')[0], role: CONFIG.AUTO_ROLE };
    cache.put(key, JSON.stringify(user), CONFIG.USER_CACHE_SECONDS);
    return { user: user, email: email, reason: '' };
  }
  return { user: null, email: email, reason: reason };
}

/** เพิ่มคนที่ล็อกอิน Google เข้าชีต Users อัตโนมัติ (ถ้าเปิด AUTO_REGISTER) */
function autoRegister_(sh, email) {
  if (!CONFIG.AUTO_REGISTER) return false;
  if (CONFIG.AUTO_DOMAIN && email.split('@')[1] !== CONFIG.AUTO_DOMAIN.toLowerCase()) return false;
  const lock = LockService.getScriptLock();
  try { lock.waitLock(10000); } catch (e) { return false; }
  try {
    const last = sh.getLastRow();
    const have = last < 2 ? [] : sh.getRange(2, 1, last - 1, 1).getValues().map(r => String(r[0]).toLowerCase().trim());
    if (have.indexOf(email) < 0) sh.appendRow([email, email.split('@')[0], CONFIG.AUTO_ROLE, true]);
    return true;
  } catch (e) { console.error('autoRegister', e); return false; }
  finally { lock.releaseLock(); }
}

function getCurrentUser_() { return resolveUser_().user; }

function requireRole_(minRole) {
  const r = resolveUser_();
  if (!r.user) {
    const msg = {
      noemail: 'ระบบอ่านอีเมลของคุณไม่ได้ — ให้ล็อกอินด้วยบัญชี Google แล้วเปิดลิงก์ใหม่ (ถ้ายังไม่ได้ ให้ผู้ดูแลตั้งค่า Deploy เป็น "Execute as: User accessing the web app")',
      nosheet: 'ยังไม่มีชีต ' + CONFIG.USERS_SHEET + ' หรือยังไม่มีรายชื่อ กรุณาให้ผู้ดูแลรันฟังก์ชัน setupAccessControl',
      notfound: 'อีเมล ' + r.email + ' ยังไม่อยู่ในรายชื่อผู้ใช้ กรุณาส่งอีเมลนี้ให้ผู้ดูแลระบบเพื่อเพิ่มในชีต ' + CONFIG.USERS_SHEET,
      inactive: 'บัญชี ' + r.email + ' ถูกระงับการใช้งาน (คอลัมน์ active เป็น FALSE) กรุณาติดต่อผู้ดูแลระบบ',
      badrole: 'บัญชี ' + r.email + ' ระบุสิทธิ์ (role) ในชีต ' + CONFIG.USERS_SHEET + ' ไม่ถูกต้อง ต้องเป็น viewer, editor หรือ admin'
    }[r.reason];
    throw new Error(msg);
  }
  if (ROLE_LEVEL[r.user.role] < ROLE_LEVEL[minRole]) {
    throw new Error('สิทธิ์ของคุณ (' + r.user.role + ') ไม่เพียงพอสำหรับการทำรายการนี้');
  }
  return r.user;
}

/**
 * เพิ่มเจ้าหน้าที่หลายคนในครั้งเดียว: ใส่รายชื่อใน NEW_STAFF แล้วรัน addStaff ใน Apps Script Editor
 * (หรือพิมพ์เพิ่มในชีต Users เองก็ได้ — คอลัมน์ email, name, role, active)
 */
const NEW_STAFF = [
  // ['somchai@gmail.com', 'สมชาย', 'editor'],
  // ['somying@gmail.com', 'สมหญิง', 'editor'],
];

function addStaff() {
  requireRole_('admin');
  const sh = getSS_().getSheetByName(CONFIG.USERS_SHEET);
  if (!sh) throw new Error('ยังไม่มีชีต ' + CONFIG.USERS_SHEET + ' ให้รัน setupAccessControl ก่อน');
  const have = sh.getLastRow() < 2 ? [] :
    sh.getRange(2, 1, sh.getLastRow() - 1, 1).getValues().map(r => String(r[0]).toLowerCase().trim());
  let added = 0;
  NEW_STAFF.forEach(p => {
    const email = String(p[0] || '').toLowerCase().trim();
    if (!email || have.indexOf(email) >= 0) return;
    const role = parseRole_(p[2] || 'editor') || 'editor';
    sh.appendRow([email, p[1] || email, role, true]);
    have.push(email); added++;
  });
  clearUserCache_();
  return 'เพิ่มเจ้าหน้าที่ ' + added + ' คน (ข้ามที่มีอยู่แล้ว ' + (NEW_STAFF.length - added) + ')';
}

/** หน้าเว็บเรียกตอนเปิด เพื่อรู้ว่าผู้ใช้คือใครและทำอะไรได้บ้าง */
function getMe() {
  const u = requireRole_('viewer');
  return {
    email: u.email, name: u.name, role: u.role,
    canWrite: ROLE_LEVEL[u.role] >= ROLE_LEVEL.editor,
    canDelete: u.role === 'admin'
  };
}

function audit_(user, action, target, detail) {
  try {
    const sh = getSS_().getSheetByName(CONFIG.AUDIT_SHEET);
    if (!sh) return;
    let d = String(detail == null ? '' : detail).slice(0, 1000);
    if (/^[=+\-@]/.test(d)) d = "'" + d;   // กันข้อความที่ขึ้นต้นด้วยอักขระสูตร
    sh.appendRow([new Date(), user.email, action, target, d]);
  } catch (e) { console.error('audit fail', e); }
}

/**
 * รันมือครั้งเดียวจาก Apps Script Editor: สร้างชีต Users + AuditLog
 * และใส่อีเมลของคนที่กดรันเป็น admin คนแรก
 */
function setupAccessControl() {
  const ss = SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID);
  const ensure = (name, headers) => {
    let sh = ss.getSheetByName(name);
    if (!sh) sh = ss.insertSheet(name);
    if (sh.getLastRow() === 0) {
      sh.getRange(1, 1, 1, headers.length).setValues([headers]);
      sh.setFrozenRows(1);
    }
    return sh;
  };
  const users = ensure(CONFIG.USERS_SHEET, USERS_HEADERS);
  ensure(CONFIG.AUDIT_SHEET, AUDIT_HEADERS);
  const me = (Session.getEffectiveUser().getEmail() || '').toLowerCase();
  if (me && users.getLastRow() < 2) users.appendRow([me, 'ผู้ดูแลระบบ', 'admin', true]);
  clearUserCache_();
  return 'สร้างชีต ' + CONFIG.USERS_SHEET + ' และ ' + CONFIG.AUDIT_SHEET + ' แล้ว (admin: ' + (me || 'ไม่พบอีเมล') + ')';
}

/** ล้างแคชสิทธิ์ผู้ใช้ทันทีหลังแก้ชีต Users (รันมือ) */
function clearUserCache() {
  clearUserCache_();
  return 'ล้างแคชสิทธิ์แล้ว';
}

function clearUserCache_() {
  const sh = SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID).getSheetByName(CONFIG.USERS_SHEET);
  if (!sh || sh.getLastRow() < 2) return;
  const cache = CacheService.getScriptCache();
  sh.getRange(2, 1, sh.getLastRow() - 1, 1).getValues().flat().filter(String)
    .forEach(e => cache.remove('user:' + String(e).toLowerCase().trim()));
}

/* ============================================================
 * ชีตและการแปลงข้อมูล
 * ============================================================ */

let SS_ = null;
function getSS_() {
  if (!SS_) SS_ = SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID);
  return SS_;
}

/** ชีตลำดับที่ si ใน CONFIG.SHEET_NAMES (0 = ชีตหลัก สร้างให้ถ้ายังไม่มี / ชีตอื่นถ้าไม่มีจะคืน null) */
function getSheetAt_(si) {
  si = si || 0;
  const name = CONFIG.SHEET_NAMES[si];
  if (!name) return null;
  const ss = getSS_();
  let sh = ss.getSheetByName(name);
  if (!sh) {
    if (si > 0) return null;
    sh = ss.insertSheet(name);
  }
  if (sh.getMaxColumns() < HEADERS.length) {
    try { sh.insertColumnsAfter(sh.getMaxColumns(), HEADERS.length - sh.getMaxColumns()); }
    catch (e) { throw new Error('ชีต ' + name + ' มีคอลัมน์ไม่พอ กรุณาเพิ่มคอลัมน์ว่างต่อจาก src ในชีต แล้วใส่หัวคอลัมน์ reason'); }
  }
  const cache = CacheService.getScriptCache();
  const hk = si === 0 ? K_HDR : K_HDR + '_' + si;
  if (sh.getLastRow() === 0) {
    sh.getRange(1, 1, 1, HEADERS.length).setValues([HEADERS]);
    sh.setFrozenRows(1);
    cache.remove(hk);
  } else if (!cache.get(hk)) {
    const current = sh.getRange(1, 1, 1, HEADERS.length).getDisplayValues()[0];
    const bad = HEADERS.some((h, i) => current[i] !== h);
    if (bad) throw new Error('หัวตารางในชีต ' + name + ' ไม่ตรงกับระบบ กรุณาตรวจสอบแถวที่ 1 ให้เป็น: ' + HEADERS.join(', ') + ' (ต้องลบคอลัมน์ history, createdAt, updatedAt ออกจากชีตแล้ว)');
    cache.put(hk, '1', 3600);
  }
  return sh;
}

function getSheet_() { return getSheetAt_(0); }

/** ปีค.ศ. ของชีตลำดับที่ si จากเลข พ.ศ. ท้ายชื่อชีต เช่น คำร้อง2568 → 2025 */
function sheetYear_(si) {
  const m = String(CONFIG.SHEET_NAMES[si] || '').match(/(\d{4})\s*$/);
  return m ? String(+m[1] - 543) : '';
}

/** รายการใหม่วันที่ปีไหน ลงชีตของปีนั้น (ถ้ามีชีตปีนั้น) ไม่เช่นนั้นลงชีตหลัก */
function pickSheetIndex_(date) {
  const y = dateString_(date).slice(0, 4);
  for (let i = 0; i < CONFIG.SHEET_NAMES.length; i++) {
    if (sheetYear_(i) === y && getSheetAt_(i)) return i;
  }
  return 0;
}

const DATE_MEMO_ = {};
function dateString_(v) {
  if (v instanceof Date) {
    const t = v.getTime();
    if (isNaN(t)) return '';
    let s = DATE_MEMO_[t];
    if (!s) { s = Utilities.formatDate(v, CONFIG.TIMEZONE, 'yyyy-MM-dd'); DATE_MEMO_[t] = s; }
    return s;
  }
  const s = String(v == null ? '' : v).trim();
  const m = s.match(/^(\d{4}-\d{2}-\d{2})/);
  return m ? m[1] : s;
}

function isBlank_(v) {
  return v === '' || v == null || String(v).trim() === '';
}

/** ประเภท/สาเหตุเก็บในช่องเดียวคั่นด้วยจุลภาค เช่น "รอเซ็น, รอตรวจ" → ['รอเซ็น','รอตรวจ'] */
function splitMulti_(v) {
  let t = String(v == null ? '' : v);
  if (t.indexOf(',') < 0) { const x = t.trim(); return x ? [x] : []; }
  // ตัวเลือกที่มี , อยู่ในชื่อ (ใน REASON_LIST) ต้องไม่ถูกตัดเป็น 2 รายการ
  const keep = [];
  COMMA_OPTS_.forEach(o => { if (t.indexOf(o) >= 0) { t = t.split(o).join('\u0001' + keep.length + '\u0001'); keep.push(o); } });
  return t.split(',').map(x => x.trim().replace(/\u0001(\d+)\u0001/g, (m, i) => keep[+i])).filter(Boolean);
}

function joinMulti_(arr) {
  return arr.filter((x, i) => arr.indexOf(x) === i).join(', ');
}

/** พารามิเตอร์ตัวกรองจากหน้าเว็บ: รับทั้งอาร์เรย์และข้อความเดียว */
function toArr_(v) {
  if (Array.isArray(v)) return v.map(x => String(x)).filter(Boolean);
  const s = String(v == null ? '' : v);
  return s ? [s] : [];
}

/** แถวเต็ม (17 คอลัมน์) → object สำหรับหน้ารายละเอียด/แก้ไข */
function rowToObject_(row) {
  const o = {};
  HEADERS.forEach((h, i) => { o[h] = row[i] == null ? '' : row[i]; });
  o.date = dateString_(o.date);
  o.id = String(o.ID || '');
  return o;
}

function objectToRow_(o) {
  return HEADERS.map(h => {
    let v = h === 'ID' ? (o.ID || o.id || '') : o[h];
    if (h === 'date') v = dateString_(v);
    if (h === 'type' || h === 'reason') v = joinMulti_(splitMulti_(Array.isArray(v) ? v.join(',') : v));
    return v == null ? '' : v;
  });
}

/** แถวจากชีต/แถวเต็ม → แถวย่อ (16 ค่า) ที่เก็บใน Cache โดยไม่มี Date */
function toListRow_(row, rowNo) {
  const a = new Array(LIST_COLS + 2);
  for (let i = 0; i < LIST_COLS; i++) {
    let v = row[i];
    if (v instanceof Date) v = dateString_(v);
    a[i] = v == null ? '' : v;
  }
  a[0] = String(a[0]);
  a[COL.date] = dateString_(a[COL.date]);
  a[LIST_COLS] = rowNo || 0;   // เลขแถวในชีต ใช้เป็นตัวช่วยหาแถวให้เร็ว (ตรวจซ้ำก่อนใช้เสมอ)
  a[LIST_COLS + 1] = String(row[COL.reason] == null ? '' : row[COL.reason]).slice(0, 200);   // สาเหตุ (หลายรายการคั่นด้วยจุลภาค ตัดไม่เกิน 200 ตัวอักษร)
  return a;
}

/** แถวย่อ → object ที่ส่งให้หน้าเว็บ */
function listRowToObject_(r) {
  const o = {};
  for (let i = 0; i < LIST_COLS; i++) o[HEADERS[i]] = r[i] == null ? '' : r[i];
  o.id = String(r[0]);
  o.rn = r[LIST_COLS] || 0;
  o.reason = r[LIST_COLS + 1] || '';
  return o;
}

/* ============================================================
 * Lock
 * ============================================================ */

function withLock_(fn) {
  const lock = LockService.getScriptLock();
  try { lock.waitLock(CONFIG.LOCK_WAIT_MS); }
  catch (e) { throw new Error('ระบบกำลังบันทึกข้อมูลของผู้ใช้อื่นอยู่ กรุณาลองอีกครั้ง'); }
  try { return fn(); } finally { lock.releaseLock(); }
}

/* ============================================================
 * Cache
 * ============================================================ */

function chunkKey_(token, i) { return 'rq_' + token + '_' + i; }

function metaKey_(si) { return si ? K_META + '_' + si : K_META; }

function bumpGen_() {
  const g = Utilities.getUuid();
  CacheService.getScriptCache().put(K_GEN, g, 21600);
  return g;
}

/** ล้าง Cache รายการทั้งหมด (อ่านชีตใหม่ในครั้งถัดไป) */
function invalidate_() {
  const cache = CacheService.getScriptCache();
  CONFIG.SHEET_NAMES.forEach((n, i) => cache.remove(metaKey_(i)));
  cache.remove(K_OPTS);
  bumpGen_();
}

function readState_(cache, si) {
  const mk = metaKey_(si);
  const got = cache.getAll([mk, K_GEN]);
  let meta = null;
  if (got[mk]) { try { meta = JSON.parse(got[mk]); } catch (e) { meta = null; } }
  if (meta && !(meta.exp > Date.now())) meta = null;
  return { meta: meta, gen: got[K_GEN] || '' };
}

function readChunks_(cache, meta) {
  const count = Math.ceil(meta.n / meta.s);
  if (!count) return [];
  const keys = [];
  for (let i = 0; i < count; i++) keys.push(chunkKey_(meta.t, i));
  const parts = cache.getAll(keys);
  const out = [];
  try {
    for (let k = 0; k < keys.length; k++) {
      const s = parts[keys[k]];
      if (!s) return null;
      const arr = JSON.parse(s);
      for (let j = 0; j < arr.length; j++) if (arr[j]) out.push(arr[j]);
    }
  } catch (e) { return null; }
  return out;
}

/** เก็บรายการลง Cache เป็นก้อน คืน token ของชุดที่เก็บ (ว่างถ้าเก็บไม่ได้) */
function cacheRows_(rows, gen0, cache, si) {
  // ถ้ามีการเขียนเกิดขึ้นระหว่างที่เรากำลังอ่านชีต ข้อมูลที่อ่านมาอาจเก่าแล้ว จึงไม่เก็บ
  if ((cache.get(K_GEN) || '') !== gen0) return '';
  for (let a = 0; a < CONFIG.CHUNK_ROWS.length; a++) {
    const size = CONFIG.CHUNK_ROWS[a];
    const parts = [];
    let ok = true;
    for (let i = 0; i < rows.length; i += size) {
      const s = JSON.stringify(rows.slice(i, i + size));
      if (s.length > CONFIG.MAX_CHUNK_CHARS) { ok = false; break; }
      parts.push(s);
    }
    if (!ok) continue;
    const token = Utilities.getUuid().slice(0, 8);
    const map = {};
    parts.forEach((s, k) => { map[chunkKey_(token, k)] = s; });
    if (parts.length) cache.putAll(map, CONFIG.CACHE_SECONDS);
    const meta = { t: token, n: rows.length, s: size, exp: Date.now() + CONFIG.CACHE_SECONDS * 1000 };
    cache.put(metaKey_(si), JSON.stringify(meta), CONFIG.CACHE_SECONDS);
    return token;
  }
  return '';
}

/**
 * แก้แถวเดียวใน Cache ให้ตรงกับชีต (เรียกภายใน Lock หลังเขียนชีตแล้ว)
 * rowNo = เลขแถวในชีต, isAppend = เป็นแถวใหม่ต่อท้าย
 */
function patchCacheRow_(rowNo, listRow, isAppend, si) {
  const cache = CacheService.getScriptCache();
  const mk = metaKey_(si);
  const st = readState_(cache, si), meta = st.meta;
  if (!meta) return;
  const pos = rowNo - 2;
  if (pos < 0 || (isAppend ? pos !== meta.n : pos >= meta.n)) { cache.remove(mk); return; }
  const ttl = Math.floor((meta.exp - Date.now()) / 1000);
  if (ttl < 10) { cache.remove(mk); return; }
  const k = Math.floor(pos / meta.s), i = pos % meta.s, key = chunkKey_(meta.t, k);
  let arr = [];
  if (!(isAppend && i === 0)) {
    const txt = cache.get(key);
    if (!txt) { cache.remove(mk); return; }
    try { arr = JSON.parse(txt); } catch (e) { cache.remove(mk); return; }
  }
  if (isAppend ? arr.length !== i : i >= arr.length) { cache.remove(mk); return; }
  arr[i] = listRow;
  const s = JSON.stringify(arr);
  if (s.length > CONFIG.MAX_CHUNK_CHARS + 3000) { cache.remove(mk); return; }
  cache.put(key, s, ttl);
  if (isAppend) {
    meta.n += 1;
    cache.put(mk, JSON.stringify(meta), ttl);
  }
}

/* ============================================================
 * อ่านรายการ (ผ่าน Cache)
 * ============================================================ */

function readSheetValues_(sh) {
  const last = sh.getLastRow();
  if (last < 2) return [];
  return sh.getRange(2, 1, last - 1, HEADERS.length).getValues();
}

function needsId_(r) {
  if (!isBlank_(r[0])) return false;
  for (let i = 1; i < 14; i++) if (!isBlank_(r[i])) return true;
  return false;
}

function rebuildList_(gen0, cache, si) {
  si = si || 0;
  const sh = getSheetAt_(si);
  if (!sh) {   // ชีตของปีนี้ยังไม่มี: เก็บเป็นรายการว่าง
    return { rows: [], t: cacheRows_([], gen0, cache, si), gen: gen0 };
  }
  let values = readSheetValues_(sh);
  if (values.some(needsId_)) {
    try { fillMissingIds_(si); } catch (e) { /* ถ้าติด Lock ให้ข้ามแถวที่ยังไม่มี ID ไปก่อน */ }
    values = readSheetValues_(sh);
  }
  // เก็บตำแหน่งแถวให้ตรงกับชีต (แถวว่างเป็น null) เพื่อให้แก้ใน Cache ได้ตรงตำแหน่ง
  const rows = values.map((r, i) => isBlank_(r[0]) ? null : toListRow_(r, encRn_(si, i + 2)));
  const token = cacheRows_(rows, gen0, cache, si);
  return { rows: rows.filter(Boolean), t: token, gen: gen0 };
}

/** รวมรายการจากทุกชีตปี (แต่ละชีตมี Cache ของตัวเอง) */
function getList_() {
  const cache = CacheService.getScriptCache();
  const rows = [], toks = [];
  let gen = null;
  for (let si = 0; si < CONFIG.SHEET_NAMES.length; si++) {
    const st = readState_(cache, si);
    if (gen === null) gen = st.gen;
    let part = null;
    if (st.meta) {
      const r = readChunks_(cache, st.meta);
      if (r) part = { rows: r, t: st.meta.t };
    }
    if (!part) part = rebuildList_(st.gen, cache, si);
    for (let k = 0; k < part.rows.length; k++) rows.push(part.rows[k]);
    toks.push(part.t);
  }
  return { rows: rows, t: toks.every(Boolean) ? toks.join('.') : '', gen: gen || '' };
}

/** โทเคนรวมของ Cache ทุกชีต (ว่างถ้ามีชีตใดไม่มี Cache) ใช้เป็นกุญแจแคช Dashboard */
function listToken_(cache) {
  const keys = [K_GEN];
  CONFIG.SHEET_NAMES.forEach((n, i) => keys.push(metaKey_(i)));
  const got = cache.getAll(keys);
  const toks = [];
  for (let i = 0; i < CONFIG.SHEET_NAMES.length; i++) {
    let meta = null;
    try { meta = JSON.parse(got[metaKey_(i)]); } catch (e) { return null; }
    if (!meta || !(meta.exp > Date.now())) return null;
    toks.push(meta.t);
  }
  return { t: toks.join('.'), gen: got[K_GEN] || '' };
}

/* ============================================================
 * Dashboard
 * ============================================================ */

/** มีผลตรวจแล้ว = ผ่าน หรือ ไม่ผ่าน (ยังรับค่าเก่าที่มีคำว่า "เสร็จ" เป็นตรวจแล้ว) นอกนั้นนับเป็นรอผลตรวจ */
function isDoneStatus_(st) {
  const t = String(st == null ? '' : st).trim();
  return t === 'ผ่าน' || t === 'ไม่ผ่าน' || t.indexOf('เสร็จ') >= 0;
}

function computeDashboard_(list, year, requestedMonth) {
  const rows = list.map(r => ({
    date: r[COL.date],
    type: r[COL.type] || '',
    status: String(r[COL.status] || ''),
    phase: r[COL.phase] || ''
  }));
  const isDone = r => isDoneStatus_(r.status);
  const y = String(year || '');
  const filtered = y ? rows.filter(r => r.date.indexOf(y) === 0) : rows;
  const years = [...new Set(rows.map(r => r.date.slice(0, 4)).filter(x => /^\d{4}$/.test(x)))].sort();
  const months = [...new Set(filtered.map(r => r.date.slice(0, 7)).filter(x => /^\d{4}-\d{2}$/.test(x)))].sort();
  const today = Utilities.formatDate(new Date(), CONFIG.TIMEZONE, 'yyyy-MM-dd');
  const yesterday = Utilities.formatDate(new Date(Date.now() - 86400000), CONFIG.TIMEZONE, 'yyyy-MM-dd');
  const month = months.indexOf(String(requestedMonth)) >= 0 ? String(requestedMonth) :
    (months.indexOf(today.slice(0, 7)) >= 0 ? today.slice(0, 7) : (months[months.length - 1] || today.slice(0, 7)));
  const monthRows = filtered.filter(r => r.date.slice(0, 7) === month);
  const prevMonth = months[months.indexOf(month) - 1];
  const countBy = (arr, fn) => arr.reduce((a, r) => { const k = fn(r); if (k) a[k] = (a[k] || 0) + 1; return a; }, {});
  // คำร้องที่มีหลายประเภท นับในทุกประเภทที่มี (ผลรวมแยกประเภทจึงอาจมากกว่าจำนวนคำร้อง)
  const countTypes = arr => arr.reduce((a, r) => { splitMulti_(r.type).forEach(t => { a[t] = (a[t] || 0) + 1; }); return a; }, {});
  const byMonth = countBy(filtered, r => r.date.slice(0, 7));
  const byYear = countBy(rows, r => r.date.slice(0, 4));
  const byDay = countBy(monthRows, r => r.date);
  const monthSummary = months.map(m => {
    const a = filtered.filter(r => r.date.slice(0, 7) === m);
    return { key: m, total: a.length, open: a.filter(r => !isDone(r)).length, types: countTypes(a) };
  });
  return {
    total: filtered.length,
    allTotal: rows.length,
    years: years,
    months: months,
    month: month,
    today: filtered.filter(r => r.date === today).length,
    yesterday: filtered.filter(r => r.date === yesterday).length,
    monthTotal: monthRows.length,
    previousMonthTotal: prevMonth ? (byMonth[prevMonth] || 0) : null,
    open: filtered.filter(r => !isDone(r)).length,
    doneMonth: monthRows.filter(isDone).length,
    byMonth: byMonth,
    byYear: byYear,
    byDay: byDay,
    types: countTypes(monthRows),
    phases: countBy(monthRows.filter(r => r.phase && r.phase !== '-'), r => r.phase),
    monthSummary: monthSummary,
    yearSummary: years.map(yy => ({ key: yy, total: byYear[yy] || 0 }))
  };
}

/**
 * รายการตัวเลือก "ประเภท" และ "สาเหตุ" — รวมจากทุกแหล่งเข้าด้วยกัน (ตัดซ้ำ เรียงตามที่พบ)
 *  1) แท็บชื่อ CONFIG.OPTIONS_SHEET ('ตัวเลือก') แถว 1 = หัวคอลัมน์ type / reason (หรือ ประเภท / สาเหตุ) แถวถัดลงไป = รายการ
 *  2) dropdown ที่ตั้งไว้ในคอลัมน์ type / reason ของ "ทุกชีตปี" (สุ่มตรวจแถวบน/ล่าง/ทุก 250 แถว) ทุกชุดรายการที่ไม่ซ้ำกันจะถูกรวม
 * ถ้าไม่เจอเลย จะส่งรายการว่าง หน้าเว็บจะใช้รายการตั้งต้นในไฟล์ index.html (TYPES / REASONS) แทน
 * หมายเหตุ: dropdown แบบ "คอลัมน์ชนิดตาราง (Table column type)" ของ Google Sheets Apps Script อ่านไม่ได้ → ใช้วิธีที่ 1
 */
function readOptionsInfo_() {
  const out = { types: [], reasons: [], source: { type: '-', reason: '-' }, notes: [] };
  const clean = list => {
    const seen = {}, res = [];
    list.forEach(x => {
      const v = String(x == null ? '' : x).trim();
      if (!v || seen[v]) return;
      if (v.indexOf(',') >= 0 && REASON_LIST.indexOf(v) < 0) { out.notes.push('ข้ามตัวเลือกที่มีเครื่องหมายจุลภาค: ' + v); return; }
      seen[v] = 1; res.push(v);
    });
    return res;
  };
  const ss = getSS_();
  const T = SpreadsheetApp.DataValidationCriteria;
  const targets = [
    { key: 'types', col: 'type', names: ['type', 'ประเภท'], fixed: TYPE_LIST },
    { key: 'reasons', col: 'reason', names: ['reason', 'สาเหตุ'] }
  ];
  let optSh = null;
  try { optSh = ss.getSheetByName(CONFIG.OPTIONS_SHEET); } catch (e) { /* ไม่มี */ }
  const optVals = (optSh && optSh.getLastRow() >= 1 && optSh.getLastColumn() >= 1)
    ? optSh.getRange(1, 1, optSh.getLastRow(), optSh.getLastColumn()).getDisplayValues() : null;

  targets.forEach(t => {
    const single = t.col, all = [], srcs = [];

    // 1) แท็บตัวเลือก
    if (optVals) {
      const hdr = optVals[0].map(x => String(x).trim().toLowerCase());
      const ci = hdr.findIndex(h => t.names.indexOf(h) >= 0);
      if (ci >= 0) {
        const l = optVals.slice(1).map(r => r[ci]).filter(v => String(v).trim());
        if (l.length) { l.forEach(v => all.push(v)); srcs.push('แท็บ ' + CONFIG.OPTIONS_SHEET + ' (' + l.length + ')'); }
      }
    }

    // ประเภท: ไม่มีในแท็บ = ใช้ TYPE_LIST ในโค้ด (ไม่สแกน dropdown ของชีต)
    if (t.fixed) {
      if (!all.length) { t.fixed.forEach(v => all.push(v)); srcs.push('รายการในโค้ด (TYPE_LIST)'); }
      const r0 = clean(all);
      if (r0.length) { out[t.key] = r0; out.source[single] = srcs.join(' + '); }
      return;
    }

    // 2) dropdown ในทุกชีตปี รวมทุกชุดรายการที่ไม่ซ้ำกัน
    for (let si = 0; si < CONFIG.SHEET_NAMES.length; si++) {
      let sh = null;
      try { sh = getSheetAt_(si); } catch (e) { out.notes.push(single + ': เปิดชีตลำดับ ' + si + ' ไม่ได้ — ' + e.message); }
      if (!sh) continue;
      try {
        const n = Math.max(1, sh.getMaxRows() - 1);
        const dvs = sh.getRange(2, COL[t.col] + 1, n, 1).getDataValidations();
        // ไม่ตรวจทุกแถว (ชีตหลายหมื่นแถวจะช้าจนหมดเวลา): ตรวจ 60 แถวบน, 60 แถวล่างของข้อมูล, 60 แถวท้ายชีต และสุ่มทุก 250 แถว
        const pick = {};
        const addRange = (from, to) => { for (let i = Math.max(0, from); i <= Math.min(dvs.length - 1, to); i++) pick[i] = 1; };
        addRange(0, 59);
        addRange(sh.getLastRow() - 2 - 59, sh.getLastRow() - 2 + 5);
        addRange(dvs.length - 60, dvs.length - 1);
        for (let i = 0; i < dvs.length; i += 250) pick[i] = 1;
        const seen = {};
        let found = 0, warned = false;
        Object.keys(pick).map(Number).sort((a, b) => a - b).forEach(i => {
          const dv = dvs[i][0];
          if (!dv) return;
          const kind = dv.getCriteriaType(), vals = dv.getCriteriaValues();
          let list = null, key = null;
          if (kind === T.VALUE_IN_LIST) { list = vals[0]; key = 'L:' + list.join('\u0001'); }
          else if (kind === T.VALUE_IN_RANGE && vals[0] && vals[0].getA1Notation) { key = 'R:' + vals[0].getSheet().getName() + '!' + vals[0].getA1Notation(); }
          if (!key) {
            if (!warned) { out.notes.push(single + ' [' + sh.getName() + ']: แถว ' + (i + 2) + ' มี validation ชนิด ' + kind + ' (ไม่ใช่รายการ dropdown)'); warned = true; }
            return;
          }
          if (seen[key]) return;
          seen[key] = 1; found++;
          if (!list) list = [].concat.apply([], vals[0].getValues());   // อ่านค่าจากช่วงเซลล์เฉพาะชุดที่ยังไม่เคยเจอ
          list.forEach(v => all.push(v));
        });
        if (found) srcs.push('dropdown ในชีต ' + sh.getName() + ' (' + found + ' ชุด)');
        else out.notes.push(single + ' [' + sh.getName() + ']: ไม่พบ dropdown ที่อ่านได้ในคอลัมน์นี้');
      } catch (e) { out.notes.push(single + ' [' + sh.getName() + ']: อ่าน dropdown ไม่ได้ — ' + e.message); }
    }

    const res = clean(all);
    if (res.length) { out[t.key] = res; out.source[single] = srcs.join(' + '); }
  });
  return out;
}

function getOptions_() {
  const cache = CacheService.getScriptCache();
  const hit = cache.get(K_OPTS);
  if (hit) { try { return JSON.parse(hit); } catch (e) { /* อ่านใหม่ */ } }
  const out = { types: [], reasons: [] };
  try {
    const info = readOptionsInfo_();
    out.types = info.types.slice();
    out.reasons = info.reasons.length ? info.reasons : REASON_LIST;
  } catch (e) { console.error('getOptions', e); }
  // ถ้าอ่านได้ครบ เก็บตามปกติ ถ้ายังว่าง เก็บแค่สั้น ๆ จะได้ลองใหม่เร็ว
  const full = out.types.length && out.reasons.length;
  try { cache.put(K_OPTS, JSON.stringify(out), full ? CONFIG.CACHE_SECONDS : 20); } catch (e) { /* ไม่เป็นไร */ }
  return out;
}

/** รันมือใน Apps Script Editor แล้วดู Execution log: บอกว่าอ่านรายการประเภท/สาเหตุได้จากไหน ได้อะไรบ้าง */
function testOptions() {
  requireRole_('admin');
  CacheService.getScriptCache().remove(K_OPTS);
  const info = readOptionsInfo_();
  const txt = [
    'ชื่อแท็บที่อ่าน: ' + CONFIG.SHEET_NAME,
    'ประเภท  [' + info.source.type + ']: ' + (info.types.join(' | ') || '(ว่าง)'),
    'สาเหตุ  [' + info.source.reason + ']: ' + (info.reasons.join(' | ') || '(ว่าง)'),
    info.notes.length ? 'หมายเหตุ:\n - ' + info.notes.join('\n - ') : ''
  ].filter(Boolean).join('\n');
  console.log(txt);
  return txt;
}

function getOptions() {
  requireRole_('viewer');
  return getOptions_();
}

/** Dashboard ส่งกลับเฉพาะตัวเลขสรุปและข้อมูลกราฟ ไม่ส่งข้อมูลทุกรายการ */
function getDashboardData(year, requestedMonth) {
  requireRole_('viewer');
  const cache = CacheService.getScriptCache();
  const y = String(year || ''), m = String(requestedMonth || '');
  const tk = listToken_(cache);
  const keyOf = (t, gen) => ['rq_d', t, gen, y, m].join('_');
  if (tk) {
    const hit = cache.get(keyOf(tk.t, tk.gen));
    if (hit) { try { return JSON.parse(hit); } catch (e) { /* คำนวณใหม่ */ } }
  }
  const L = getList_();
  const result = computeDashboard_(L.rows, y, m);
  if (L.t) {
    try { cache.put(keyOf(L.t, L.gen), JSON.stringify(result), CONFIG.CACHE_SECONDS); } catch (e) { /* ใหญ่เกินไป ไม่ต้องเก็บ */ }
  }
  return result;
}

/* ============================================================
 * รายการคำร้อง (ค้นหา/กรอง/แบ่งหน้า)
 * ============================================================ */

function getRequestsPage(params) {
  requireRole_('viewer');
  params = params || {};
  const page = Math.max(0, Number(params.page) || 0);
  const pageSize = Math.min(CONFIG.MAX_PAGE_SIZE, Math.max(1, Number(params.pageSize) || CONFIG.DEFAULT_PAGE_SIZE));
  const q = String(params.q || '').trim().toLocaleLowerCase();
  const year = String(params.year || '');
  const month = String(params.month || '');
  const date = String(params.date || '');
  const typesF = toArr_(params.type);       // เลือกหลายประเภท = มีประเภทใดประเภทหนึ่ง
  const reasonsF = toArr_(params.reason);   // เลือกหลายสาเหตุ ('__none' = ยังไม่ระบุสาเหตุ)
  const status = String(params.status || '');

  const list = getList_().rows;
  const D = COL.date, T = COL.type, S = COL.status, RS = LIST_COLS + 1;

  // รายการตัวกรอง (ปี / เดือน / ประเภท) คำนวณจากข้อมูลทั้งหมด ไม่ลดลงเมื่อเลือกตัวกรอง
  const yearSet = {}, monthSet = {}, typeSet = {}, reasonSet = {};
  list.forEach(r => {
    const d = r[D];
    if (/^\d{4}-\d{2}/.test(d)) {
      yearSet[d.slice(0, 4)] = 1;
      if (!year || d.indexOf(year) === 0) monthSet[d.slice(0, 7)] = 1;
    }
    if (r[T]) splitMulti_(r[T]).forEach(x => { typeSet[x] = 1; });
    if (r[RS]) splitMulti_(r[RS]).forEach(x => { reasonSet[x] = 1; });
  });

  const filtered = list.filter(r => {
    const d = r[D], st = String(r[S]);
    if (year && d.indexOf(year) !== 0) return false;
    if (month && d.indexOf(month) !== 0) return false;
    if (date && d !== date) return false;
    if (typesF.length) {
      const ts = splitMulti_(r[T]);
      if (!ts.some(x => typesF.indexOf(x) >= 0)) return false;
    }
    if (reasonsF.length) {
      const rs = splitMulti_(r[RS]);
      const ok = rs.length ? rs.some(x => reasonsF.indexOf(x) >= 0) : reasonsF.indexOf('__none') >= 0;
      if (!ok) return false;
    }
    if (status === '__open' && isDoneStatus_(st)) return false;
    if (status === '__done' && !isDoneStatus_(st)) return false;
    if (status === '__pass' && String(st).trim() !== 'ผ่าน') return false;
    if (status === '__fail' && String(st).trim() !== 'ไม่ผ่าน') return false;
    if (q) {
      const hay = [r[COL.no], r[COL.name], r[COL.address], r[COL.phone], r[COL.route], r[T], r[RS], st].join(' ').toLocaleLowerCase();
      if (hay.indexOf(q) < 0) return false;
    }
    return true;
  });

  const typeCounts = {};
  let openCount = 0;
  filtered.forEach(r => {
    splitMulti_(r[T]).forEach(x => { typeCounts[x] = (typeCounts[x] || 0) + 1; });
    if (!isDoneStatus_(r[S])) openCount++;
  });

  // ใหม่สุดก่อน (เรียงเฉพาะรายการที่ผ่านตัวกรองแล้ว ใช้การเทียบข้อความธรรมดาเพื่อความเร็ว)
  const N = COL.no;
  filtered.sort((a, b) => {
    if (a[D] !== b[D]) return a[D] < b[D] ? 1 : -1;
    const na = String(a[N]), nb = String(b[N]);
    return na === nb ? 0 : (na < nb ? 1 : -1);
  });

  const total = filtered.length;
  const pages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(page, pages - 1);
  return {
    rows: filtered.slice(safePage * pageSize, safePage * pageSize + pageSize).map(listRowToObject_),
    total: total,
    page: safePage,
    pageSize: pageSize,
    pages: pages,
    openCount: openCount,
    typeCounts: typeCounts,
    types: Object.keys(typeSet).sort(),
    reasons: Object.keys(reasonSet).sort(),
    years: Object.keys(yearSet).sort(),
    months: Object.keys(monthSet).sort()
  };
}

/**
 * สมุดคุมคำร้อง: ส่งรายชื่อปี + สรุปจำนวนรายเดือนของปีที่เลือก + รายการของเดือนที่เลือก ในครั้งเดียว
 * year ว่าง = ปีปัจจุบัน (หรือปีล่าสุดที่มีข้อมูล), year 'all' = ทุกปี, month 'auto' = เดือนนี้ (หรือเดือนล่าสุดที่มีข้อมูล), month ว่าง = ทั้งปี
 */
function getBookData(params) {
  requireRole_('viewer');
  const p = Object.assign({}, params || {});
  const years = getDashboardData('', '').years || [];
  const thisYear = Utilities.formatDate(new Date(), CONFIG.TIMEZONE, 'yyyy');
  let year = String(p.year || '');
  if (year === 'all') {   // ทุกปี: ไม่แยกปี/เดือน ใช้ค้นหาข้ามทุกปี
    p.year = '';
    p.month = '';
    return { year: 'all', month: '', years: years, months: [], page: getRequestsPage(p) };
  }
  if (years.indexOf(year) < 0) year = years.indexOf(thisYear) >= 0 ? thisYear : (years[years.length - 1] || thisYear);
  const dash = getDashboardData(year, '');
  let month = p.month == null ? 'auto' : String(p.month);
  if (month === 'auto') {
    const cur = Utilities.formatDate(new Date(), CONFIG.TIMEZONE, 'yyyy-MM');
    month = dash.months.indexOf(cur) >= 0 ? cur : (dash.months[dash.months.length - 1] || '');
  }
  if (month && month.indexOf(year) !== 0) month = '';
  p.year = year;
  p.month = month;
  return { year: year, month: month, years: years, months: dash.monthSummary, page: getRequestsPage(p) };
}

/* ============================================================
 * อ่าน/เพิ่ม/แก้ไข/ลบ รายการเดียว
 * ============================================================ */

/** หาแถวของคำร้องจาก ID ในทุกชีตปี hint = เลขแถวที่หน้าเว็บบอก (ลำดับชีต * RN_STRIDE + เลขแถว) คืน {sh, si, rowNo} หรือ null */
function locateRow_(id, hint) {
  const h = Number(hint) || 0;
  if (h >= 2) {
    const hsi = Math.floor(h / RN_STRIDE), hrow = h % RN_STRIDE;
    const sh = hsi < CONFIG.SHEET_NAMES.length ? getSheetAt_(hsi) : null;
    if (sh && hrow >= 2 && hrow <= sh.getLastRow() && String(sh.getRange(hrow, 1).getValue()) === String(id)) {
      return { sh: sh, si: hsi, rowNo: hrow };   // ตรงตามที่หน้าเว็บบอก ไม่ต้องไล่หา
    }
  }
  for (let si = 0; si < CONFIG.SHEET_NAMES.length; si++) {
    const sh = getSheetAt_(si);
    if (!sh) continue;
    const last = sh.getLastRow();
    if (last < 2) continue;
    const cell = sh.getRange(2, 1, last - 1, 1).createTextFinder(String(id)).matchEntireCell(true).findNext();
    if (cell) return { sh: sh, si: si, rowNo: cell.getRow() };
  }
  return null;
}

function getRequest(id, hint) {
  requireRole_('viewer');
  if (!id) throw new Error('ไม่พบรหัสคำร้อง');
  const loc = locateRow_(id, hint);
  if (!loc) throw new Error('ไม่พบคำร้องที่ต้องการ');
  const o = rowToObject_(loc.sh.getRange(loc.rowNo, 1, 1, HEADERS.length).getValues()[0]);
  o.rn = encRn_(loc.si, loc.rowNo);
  return o;
}

function nextRequestNo_(sheet, date) {
  const d = dateString_(date);
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d)) return '';
  const prefix = String((Number(d.slice(0, 4)) + 543) % 100) + d.slice(5, 7) + '-';
  const last = sheet.getLastRow();
  if (last < 2) return prefix + '0001';
  const nums = sheet.getRange(2, COL.no + 1, last - 1, 1).getDisplayValues();
  let max = 0;
  nums.forEach(r => {
    const value = String(r[0] || '');
    if (value.indexOf(prefix) === 0) {
      const n = Number(value.slice(prefix.length));
      if (isFinite(n) && n > max) max = n;
    }
  });
  return prefix + String(max + 1).padStart(4, '0');
}

function createRequest(data) {
  const user = requireRole_('editor');
  const out = withLock_(() => {
    const input = Object.assign({}, data || {});
    delete input._base;
    const si = pickSheetIndex_(input.date), sh = getSheetAt_(si), id = Utilities.getUuid();
    if (!String(input.no || '').trim()) input.no = nextRequestNo_(sh, input.date);
    const obj = Object.assign({}, input, { ID: id, id: id });
    const rowNo = sh.getLastRow() + 1;
    const row = objectToRow_(obj);
    if (rowNo > sh.getMaxRows()) sh.insertRowsAfter(sh.getMaxRows(), rowNo - sh.getMaxRows());
    copyValidationFromAbove_(sh, rowNo);   // ให้แถวใหม่มี dropdown เหมือนแถวอื่น
    sh.getRange(rowNo, 1, 1, HEADERS.length).setValues([row]);
    patchCacheRow_(rowNo, toListRow_(row, encRn_(si, rowNo)), true, si);
    const prevVer = CacheService.getScriptCache().get(K_GEN) || '';
    const ver = bumpGen_();
    return { success: true, id: id, no: input.no, ver: ver, prevVer: prevVer };
  });
  audit_(user, 'create', out.no || out.id, 'ID ' + out.id);   // บันทึกประวัติหลังปล่อย Lock เพื่อไม่ให้คนอื่นรอ
  return out;
}

function updateRequest(id, data, hint) {
  const user = requireRole_('editor');
  if (!id) throw new Error('ไม่พบรหัสคำร้อง');
  let auditText = '', auditNo = '';
  const out = withLock_(() => {
    const loc = locateRow_(id, hint);
    if (!loc) throw new Error('ไม่พบคำร้องที่ต้องการแก้ไข');
    const sh = loc.sh, si = loc.si, rowNo = loc.rowNo;
    const old = rowToObject_(sh.getRange(rowNo, 1, 1, HEADERS.length).getValues()[0]);
    const incoming = Object.assign({}, data || {});
    delete incoming._base;

    const merged = Object.assign({}, old, incoming, { ID: String(id), id: String(id) });
    const has = k => Object.prototype.hasOwnProperty.call(incoming, k);
    const changed = Object.keys(FIELD_LABELS).filter(k =>
      has(k) && String(incoming[k] == null ? '' : incoming[k]) !== String(old[k] == null ? '' : old[k]));
    const parts = [];
    if (has('status') && incoming.status !== old.status) parts.push('สถานะ: ' + (old.status || '-') + ' → ' + incoming.status);
    if (changed.length) parts.push('แก้ไข ' + changed.map(k => FIELD_LABELS[k]).join(', '));

    const row = objectToRow_(merged);
    sh.getRange(rowNo, 1, 1, HEADERS.length).setValues([row]);
    patchCacheRow_(rowNo, toListRow_(row, encRn_(si, rowNo)), false, si);
    const prevVer = CacheService.getScriptCache().get(K_GEN) || '';
    const ver = bumpGen_();
    auditText = parts.join(' · ');
    auditNo = old.no || String(id);
    return { success: true, ver: ver, prevVer: prevVer };
  });
  if (auditText) audit_(user, 'update', auditNo, auditText);   // หลังปล่อย Lock
  return out;
}

function deleteRequest(id) {
  const user = requireRole_('admin');
  if (!id) throw new Error('ไม่พบรหัสคำร้อง');
  return withLock_(() => {
    const loc = locateRow_(id);
    if (!loc) throw new Error('ไม่พบคำร้องที่ต้องการลบ');
    const sh = loc.sh, rowNo = loc.rowNo;
    const old = rowToObject_(sh.getRange(rowNo, 1, 1, HEADERS.length).getValues()[0]);
    sh.deleteRow(rowNo);
    audit_(user, 'delete', old.no || String(id), JSON.stringify({ name: old.name, date: old.date, type: old.type, status: old.status, phone: old.phone }));
    invalidate_(); // แถวด้านล่างเลื่อนขึ้น ตำแหน่งใน Cache เปลี่ยน จึงต้องอ่านใหม่
    return { success: true };
  });
}

/* ============================================================
 * ตัวช่วยสำหรับแก้ข้อมูลในชีตโดยตรง
 * ============================================================ */

/** เติม ID ให้แถวที่พิมพ์เองในชีตแล้วคอลัมน์ ID ว่าง คืนจำนวนแถวที่เติม */
function fillMissingIds() {
  requireRole_('admin');
  return fillMissingIds_();
}

function fillMissingIds_(si) {
  if (si == null) {   // ไม่ระบุชีต = ทำทุกชีตปี
    let total = 0;
    for (let i = 0; i < CONFIG.SHEET_NAMES.length; i++) {
      if (getSheetAt_(i)) total += fillMissingIds_(i);
    }
    return total;
  }
  return withLock_(() => {
    const sh = getSheetAt_(si);
    if (!sh) return 0;
    const values = readSheetValues_(sh); // อ่านใหม่ภายใน Lock เพื่อไม่ให้แถวเลื่อนระหว่างทาง
    let n = 0;
    const ids = values.map(r => {
      if (needsId_(r)) { n++; return [Utilities.getUuid()]; }
      return [r[0]];
    });
    if (n) {
      sh.getRange(2, 1, ids.length, 1).setValues(ids);
      invalidate_();
    }
    return n;
  });
}

/** ปุ่ม ↻ รีเฟรช ในหน้าเว็บเรียกใช้ฟังก์ชันนี้ */
function refreshData() {
  requireRole_('editor');
  let filled = 0;
  try { filled = fillMissingIds_(); } catch (e) { /* ไม่เป็นไร ล้าง Cache ต่อ */ }
  invalidate_();
  return { filled: filled };
}

/** รันมือครั้งเดียว: ตั้ง dropdown "ผ่าน / ไม่ผ่าน" ให้คอลัมน์ status ทั้งคอลัมน์ (ค่าเดิมที่ไม่ตรงรายการยังอยู่ ไม่ถูกลบ) */
function setupStatusDropdown() {
  requireRole_('admin');
  const sh = getSheet_();
  const rows = Math.max(1, sh.getMaxRows() - 1);
  try {
    const rule = SpreadsheetApp.newDataValidation().requireValueInList(['ผ่าน', 'ไม่ผ่าน'], true).setAllowInvalid(true).build();
    sh.getRange(2, COL.status + 1, rows, 1).setDataValidation(rule);
    return 'ตั้ง dropdown ผ่าน/ไม่ผ่าน ที่คอลัมน์ status แล้ว';
  } catch (e) {
    return 'ตั้ง dropdown ไม่ได้: ' + e.message + ' (ถ้าชีตเป็นตาราง ให้คลิกขวาหัวคอลัมน์ status > Column type > Dropdown แล้วใส่ ผ่าน และ ไม่ผ่าน)';
  }
}


/** รันมือครั้งเดียว: ตั้ง dropdown "สาเหตุ" (REASON_LIST ด้านบน) ให้คอลัมน์ reason ทั้งคอลัมน์ ค่าเดิมที่ไม่ตรงรายการยังอยู่ ไม่ถูกลบ */
function setupReasonDropdown() {
  requireRole_('admin');
  const sh = getSheet_();
  const rows = Math.max(1, sh.getMaxRows() - 1);
  try {
    const rule = SpreadsheetApp.newDataValidation().requireValueInList(REASON_LIST, true).setAllowInvalid(true).build();
    sh.getRange(2, COL.reason + 1, rows, 1).setDataValidation(rule);
    CacheService.getScriptCache().remove(K_OPTS);
    return 'ตั้ง dropdown สาเหตุ ' + REASON_LIST.length + ' รายการที่คอลัมน์ reason แล้ว';
  } catch (e) {
    return 'ตั้ง dropdown ไม่ได้: ' + e.message + ' (ถ้าชีตเป็นตาราง ให้ตั้ง dropdown ที่หัวคอลัมน์ reason เอง หรือสร้างแท็บ ตัวเลือก)';
  }
}

/** คัดลอกกฎ Data validation (dropdown) จากแถวบนมาใส่แถวที่ระบุ */
function copyValidationFromAbove_(sh, rowNo) {
  try {
    if (rowNo <= 2) return;
    sh.getRange(rowNo - 1, 1, 1, HEADERS.length)
      .copyTo(sh.getRange(rowNo, 1, 1, HEADERS.length), SpreadsheetApp.CopyPasteType.PASTE_DATA_VALIDATION, false);
  } catch (e) { console.error('copy validation', e); }
}

/**
 * รันมือครั้งเดียว: ซ่อม dropdown ให้ทุกแถวที่มีอยู่แล้ว (รวมแถวว่างด้านล่างที่จะใช้ต่อ)
 * โดยใช้แถวที่ 2 เป็นต้นแบบ ดังนั้นแถวที่ 2 ต้องมี dropdown ครบทุกคอลัมน์
 */
function fixDropdowns() {
  requireRole_('admin');
  const sh = getSheet_();
  const total = sh.getMaxRows();
  if (total < 3) return 'ไม่มีแถวให้ซ่อม';
  try {
    sh.getRange(2, 1, 1, HEADERS.length)
      .copyTo(sh.getRange(3, 1, total - 2, HEADERS.length), SpreadsheetApp.CopyPasteType.PASTE_DATA_VALIDATION, false);
  } catch (e) {
    return 'คัดลอก dropdown ไม่ได้: ' + e.message + ' (ถ้าชีตเป็นตาราง ให้ลากขอบตารางให้ครอบแถวด้านล่างแทน)';
  }
  return 'คัดลอก dropdown จากแถวที่ 2 ไปยังแถวที่ 3–' + total + ' แล้ว';
}

/**
 * กด Run ครั้งเดียว: ติดตั้งตัวดักการแก้ไขในชีต 2 ตัว
 *  - onSheetEdit  : เมื่อพิมพ์/วางข้อมูลในชีต ระบบเติม ID, เวลา และประวัติให้ทันที (เหมือนเพิ่มจากหน้าเว็บ)
 *  - onSheetChange: เมื่อเพิ่ม/ลบแถวในชีต ล้าง Cache
 * หน้าเว็บจะรู้ว่าข้อมูลเปลี่ยนภายในไม่กี่วินาที (ดู getVersion)
 */
function setupAutoRefresh() {
  requireRole_('admin');
  ScriptApp.getProjectTriggers().forEach(t => {
    const h = t.getHandlerFunction();
    if (h === 'onSheetChange' || h === 'onSheetEdit') ScriptApp.deleteTrigger(t);
  });
  const ss = SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID);
  ScriptApp.newTrigger('onSheetChange').forSpreadsheet(ss).onChange().create();
  ScriptApp.newTrigger('onSheetEdit').forSpreadsheet(ss).onEdit().create();

  // คอลัมน์ข้อความล้วน: เบอร์โทรไม่เสียเลข 0 นำหน้า และข้อความที่ขึ้นต้นด้วย = + - @ ไม่ถูกตีเป็นสูตร
  // ถ้าคอลัมน์เป็นชนิดที่กำหนดไว้ในตาราง (Table) ของ Google Sheets จะตั้งรูปแบบไม่ได้ ให้ข้ามไป
  const skipped = [];
  try {
    CONFIG.SHEET_NAMES.forEach((n, si) => {
      const sh = getSheetAt_(si);
      if (!sh) return;
      const rows = Math.max(1, sh.getMaxRows() - 1);
      ['no', 'name', 'address', 'phone', 'note'].forEach(h => {
        try { sh.getRange(2, COL[h] + 1, rows, 1).setNumberFormat('@'); }
        catch (e) { if (skipped.indexOf(h) < 0) skipped.push(h); }
      });
    });
  } catch (e) { console.error('format', e); }

  invalidate_();
  return 'ตั้งค่าเรียบร้อย: แก้ในชีตแล้วหน้าเว็บจะอัปเดตเอง และแถวที่พิมพ์เองจะได้ ID ทันที'
    + (skipped.length ? ' (ข้ามการตั้งรูปแบบข้อความของคอลัมน์ ' + skipped.join(', ') + ' เพราะเป็นคอลัมน์ชนิดตาราง)' : '');
}

function onSheetChange(e) {
  // การแก้ค่าในเซลล์ onSheetEdit จัดการเองแล้ว (อัปเดตเฉพาะแถวที่แก้) ที่เหลือ เช่น เพิ่ม/ลบแถว/คอลัมน์ ให้ล้างแคช
  if (e && (e.changeType === 'EDIT' || e.changeType === 'FORMAT')) return;
  invalidate_();
}

/** หน้าเว็บถามเลขรุ่นของข้อมูลเป็นระยะ ถ้าเปลี่ยนก็โหลดรายการใหม่ (เบามาก อ่านจาก Cache อย่างเดียว) */
function getVersion() {
  requireRole_('viewer');
  return CacheService.getScriptCache().get(K_GEN) || '';
}

/**
 * ตัวดักการแก้ไขในชีต (Installable onEdit) — ทำให้ข้อมูลที่พิมพ์ในชีตเชื่อมกับหน้าเว็บเหมือนข้อมูลที่เพิ่มจากเว็บ:
 * เติม ID ให้แถวที่พิมพ์ใหม่ แล้วอัปเดต Cache
 */
function onSheetEdit(e) {
  if (!e || !e.range || typeof e.range.getSheet !== 'function') return;
  try {
    const sh = e.range.getSheet();
    const si = CONFIG.SHEET_NAMES.indexOf(sh.getName());
    if (si < 0) return;   // แก้ในแท็บอื่นที่ไม่ใช่ชีตคำร้อง ไม่ต้องทำอะไร
    const r1 = Math.max(2, e.range.getRow());
    const r2 = e.range.getRow() + e.range.getNumRows() - 1;
    const c1 = Math.max(2, e.range.getColumn());
    const c2 = e.range.getColumn() + e.range.getNumColumns() - 1;
    if (r2 < 2 || c1 > HEADERS.length || c2 < 2) return;   // แก้เฉพาะหัวตาราง/คอลัมน์ ID ไม่ต้องทำอะไร
    const n = r2 - r1 + 1;
    if (n > 500) { invalidate_(); return; }              // วางข้อมูลก้อนใหญ่: ให้ระบบเติม ID ตอนโหลดรายการแทน

    withLock_(() => {
      const vals = sh.getRange(r1, 1, n, HEADERS.length).getValues();
      const ids = [], listRows = [];
      let needFull = false;
      vals.forEach((row, k) => {
        const hasData = row.slice(1, 14).some(v => !isBlank_(v));
        if (!hasData) { needFull = true; ids.push([row[0]]); return; }
        const newId = isBlank_(row[0]) ? Utilities.getUuid() : row[0];
        ids.push([newId]);
        const out = row.slice(); out[0] = newId;
        listRows.push([r1 + k, toListRow_(out, encRn_(si, r1 + k))]);
      });
      sh.getRange(r1, 1, n, 1).setValues(ids);
      if (needFull || listRows.length > 20) { invalidate_(); return; }
      // แก้เฉพาะแถวที่เปลี่ยนในแคช ไม่ต้องอ่านทั้งชีตใหม่
      const st = readState_(CacheService.getScriptCache(), si);
      if (st.meta) {
        let cnt = st.meta.n;
        listRows.forEach(p => {
          const isAppend = (p[0] - 2) >= cnt;
          patchCacheRow_(p[0], p[1], isAppend, si);
          if (isAppend) cnt++;
        });
      }
      bumpGen_();
    });
  } catch (err) {
    console.error('onSheetEdit', err);
    try { invalidate_(); } catch (e2) { /* ไม่เป็นไร */ }
  }
}

/* ============================================================
 * ทดสอบ
 * ============================================================ */

/**
 * ตรวจว่าตาราง (Table) ในชีตปฏิเสธคำสั่งไหนบ้าง: ทดสอบกับแถวที่ 2 ทีละคอลัมน์ โดยเขียนค่า/รูปแบบ/กฎเดิมกลับลงไป ไม่เปลี่ยนข้อมูล
 * รันแล้วดูผลใน Execution log หรือข้อความที่คืนมา
 */
function diagnoseSheet() {
  requireRole_('admin');
  const sh = getSheet_();
  if (sh.getLastRow() < 2) return 'ชีตยังไม่มีข้อมูลแถวที่ 2 ให้ทดสอบ';
  const report = [];
  const tryOp = (label, fn) => { try { fn(); } catch (e) { report.push(label + ' → ' + e.message); } };
  for (let c = 1; c <= HEADERS.length; c++) {
    const cell = sh.getRange(2, c), name = HEADERS[c - 1] + ' (คอลัมน์ ' + c + ')';
    tryOp('เขียนค่า ' + name, () => cell.setValue(cell.getValue()));
    tryOp('ตั้งรูปแบบตัวเลข ' + name, () => cell.setNumberFormat(cell.getNumberFormat()));
    tryOp('ตั้ง dropdown/validation ' + name, () => cell.setDataValidation(cell.getDataValidation()));
  }
  tryOp('เขียนทั้งแถว (แบบที่เว็บบันทึก)', () => {
    const r = sh.getRange(2, 1, 1, HEADERS.length);
    r.setValues(r.getValues());
  });
  tryOp('คัดลอก validation จากแถวบน (แบบที่เว็บเพิ่มแถว)', () => {
    sh.getRange(2, 1, 1, HEADERS.length).copyTo(sh.getRange(3, 1, 1, HEADERS.length), SpreadsheetApp.CopyPasteType.PASTE_DATA_VALIDATION, false);
  });
  const out = report.length ? 'พบคำสั่งที่ถูกปฏิเสธ:\n' + report.join('\n') : 'ไม่พบคำสั่งที่ถูกปฏิเสธในแถวที่ 2';
  console.log(out);
  return out;
}

function testConnection() {
  requireRole_('admin');
  return 'เชื่อมต่อสำเร็จ: ' + CONFIG.SHEET_NAMES.map((n, i) => {
    const sh = getSheetAt_(i);
    return sh ? sh.getName() + ' (' + Math.max(0, sh.getLastRow() - 1) + ' รายการ)' : n + ' (ไม่พบชีตนี้)';
  }).join(' · ');
}

/** กด Run แล้วดู Execution log เพื่อดูเวลาที่ใช้จริง */
function testPage() {
  requireRole_('admin');
  const t = () => Date.now();
  let a = t();
  invalidate_();
  const first = getRequestsPage({});
  Logger.log('อ่านชีตใหม่ + หน้าแรก (' + first.total + ' รายการ): ' + (t() - a) + ' ms');
  a = t(); getRequestsPage({});
  Logger.log('หน้ารายการจาก Cache: ' + (t() - a) + ' ms');
  a = t(); getRequestsPage({ q: 'ก' });
  Logger.log('ค้นหา: ' + (t() - a) + ' ms');
  a = t(); getDashboardData('', '');
  Logger.log('Dashboard (ครั้งแรก): ' + (t() - a) + ' ms');
  a = t(); getDashboardData('', '');
  Logger.log('Dashboard (จาก Cache): ' + (t() - a) + ' ms');
}
