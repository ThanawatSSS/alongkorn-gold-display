// =================================================================
// Google Apps Script (GAS) - Gold Display System (Optimized Version)
// =================================================================

const SPREADSHEET_ID = "1P8IJzEQDEYJPjV3DI6KIk_uwIL6jgzw2BcXtqswzfcY";
const LATEST_GOLD_CACHE_KEY = "latestGoldData:v1";
const DEFAULT_HEADERS = [
  "บันทึกเมื่อ", "GoldPriceID", "เวลาประกาศ", "ครั้งที่",
  "สมาคมซื้อ", "สมาคมขาย", "เทียบเมื่อวาน", "เทียบรอบก่อน", "USD/THB", "RawData"
];

/**
 * ฟังก์ชันทำงานหลัก (Auto Sync) ถูกเรียกโดย Trigger
 */
function updateGoldHybrid() {
  syncGoldPrice_();
}

/**
 * ฟังก์ชันสำหรับกดรันด้วยตัวเอง (คงชื่อเดิมสำหรับการเรียกใช้งานที่มีอยู่)
 */
function forceUpdateGoldHybrid() {
  syncGoldPrice_();
}

/**
 * ฟังก์ชันหลักในการซิงค์ข้อมูล GoldTraders ลง PriceLog
 */
function syncGoldPrice_() {
  const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
  let sheet = ss.getSheetByName("PriceLog");

  // 1. ตรวจสอบและสร้าง/อัปเดตโครงสร้างหัวตาราง (Migration)
  sheet = ensureSheetStructure_(ss, sheet);

  let rawData = null;
  let isApiSuccess = false;

  // 2. ดึงข้อมูลใหม่จาก API สมาคมฯ
  try {
    rawData = fetchGoldTraders();
    isApiSuccess = true;
  } catch (e) {
    Logger.log("Fetch GoldTraders failed: " + e.toString());
    throw e;
  }

  // 3. อ่านหัวตารางปัจจุบัน
  let headers = getCleanHeaders_(sheet);
  let lastRow = sheet.getLastRow();

  // บันทึกราคาสมาคมลง Sheet หากมีข้อมูลใหม่
  if (isApiSuccess && rawData) {
    const goldPriceIdColIdx = headers.indexOf("GoldPriceID");
    let lastPriceID = null;

    if (lastRow >= 2 && goldPriceIdColIdx !== -1) {
      lastPriceID = sheet.getRange(lastRow, goldPriceIdColIdx + 1).getValue();
    }

    if (Number(lastPriceID) !== Number(rawData.goldPriceID)) {
      const newRowData = headers.map(header => {
        switch (header) {
          case "บันทึกเมื่อ": return new Date();
          case "GoldPriceID": return rawData.goldPriceID;
          case "เวลาประกาศ": return formatThaiTime_(rawData.asTime);
          case "ครั้งที่": return rawData.seq;
          case "สมาคมซื้อ": return rawData.bL_BuyPrice;
          case "สมาคมขาย": return rawData.bL_SellPrice;
          case "เทียบเมื่อวาน": return rawData.priceChangeFromPrevDayLast;
          case "เทียบรอบก่อน": return rawData.priceChangeFromPrevRow;
          case "USD/THB": return rawData.bahtPerUSD;
          case "RawData": return JSON.stringify(rawData);
          default: return "";
        }
      });

      sheet.appendRow(newRowData);
      lastRow = sheet.getLastRow();
      headers = getCleanHeaders_(sheet);
      try {
        CacheService.getScriptCache().remove(LATEST_GOLD_CACHE_KEY);
      } catch (e) {
        Logger.log("Could not clear price cache: " + e.toString());
      }
    }
  }

  // 4. ลบข้อมูลประวัติเก่าเกิน 7 วัน
  cleanupOldLogs(sheet);

}

// ==========================================
// Helper Functions & Utilities (Optimized)
// ==========================================

/**
 * แปลงรูปแบบเวลาเป็น String ในเขตเวลาประเทศไทย (GMT+7)
 * @param {Date|string} val - ค่าที่ต้องการแปลง
 * @param {boolean} includeOffset - กำหนดให้ใส่ +07:00 ท้ายข้อความหรือไม่
 * @returns {string}
 */
function formatThaiTime_(val, includeOffset = false) {
  if (!val) return "";
  
  // กรณีเป็น Date object
  if (val instanceof Date) {
    const pattern = includeOffset 
      ? "yyyy-MM-dd'T'HH:mm:ss.SSS+07:00" 
      : "yyyy-MM-dd'T'HH:mm:ss";
    return Utilities.formatDate(val, "GMT+7", pattern);
  }

  let strVal = String(val).trim();
  
  // ถ้าเป็น ISO string มี 'Z' ต่อท้าย (UTC) ให้แปลงกลับเป็นเวลาไทย GMT+7
  if (strVal.endsWith("Z")) {
    const d = new Date(strVal);
    if (!isNaN(d.getTime())) {
      const pattern = includeOffset 
        ? "yyyy-MM-dd'T'HH:mm:ss.SSS+07:00" 
        : "yyyy-MM-dd'T'HH:mm:ss";
      return Utilities.formatDate(d, "GMT+7", pattern);
    }
  }

  return strVal;
}

/**
 * อ่านและทำความสะอาดหัวตาราง
 */
function getCleanHeaders_(sheet) {
  const lastCol = sheet.getLastColumn();
  if (lastCol === 0) return [];
  
  const headers = sheet.getRange(1, 1, 1, lastCol).getValues()[0];
  while (headers.length > 0 && !headers[headers.length - 1]) {
    headers.pop();
  }
  return headers;
}

/**
 * ตรวจสอบและสร้างโครงสร้างชีต PriceLog พร้อม Migration
 */
function ensureSheetStructure_(ss, sheet) {
  if (!sheet) {
    sheet = ss.insertSheet("PriceLog");
    sheet.appendRow(DEFAULT_HEADERS);
    return sheet;
  }

  const headers = getCleanHeaders_(sheet);
  if (headers.length === 0) {
    sheet.appendRow(DEFAULT_HEADERS);
    return sheet;
  }

  if (headers.indexOf("เทียบรอบก่อน") === -1) {
    const usdIndex = headers.indexOf("USD/THB");
    if (usdIndex !== -1) {
      const insertColIndex = usdIndex + 1;
      sheet.insertColumnBefore(insertColIndex);
      sheet.getRange(1, insertColIndex).setValue("เทียบรอบก่อน");
    } else {
      const lastColActual = headers.length;
      sheet.insertColumnBefore(lastColActual);
      sheet.getRange(1, lastColActual).setValue("เทียบรอบก่อน");
    }
  }

  return sheet;
}

/**
 * ลบข้อมูลทองเก่าที่เกิน 7 วันออกเพื่อป้องกันไม่ให้ชีตเต็ม
 */
function cleanupOldLogs(sheet) {
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) return;

  const headers = getCleanHeaders_(sheet);
  const dateColIdx = headers.indexOf("บันทึกเมื่อ");
  if (dateColIdx === -1) return;

  const dateColNum = dateColIdx + 1;
  const values = sheet.getRange(2, dateColNum, lastRow - 1, 1).getValues();

  const now = new Date().getTime();
  const cutoffTime = now - (7 * 24 * 60 * 60 * 1000); // 7 วัน

  let deleteCount = 0;
  for (let i = 0; i < values.length; i++) {
    const val = values[i][0];
    if (val) {
      const rowTime = new Date(val).getTime();
      if (!isNaN(rowTime) && rowTime < cutoffTime) {
        deleteCount++;
      } else {
        break;
      }
    } else {
      deleteCount++;
    }
  }

  if (deleteCount > 0) {
    sheet.deleteRows(2, deleteCount);
    Logger.log("ลบข้อมูลแถวเก่าเกิน 7 วันออกเรียบร้อย: " + deleteCount + " แถว");
  }
}

// ==========================================
// GoldTraders API Fetching
// ==========================================

function fetchGoldTraders() {
  const url = "https://www.goldtraders.or.th/api/GoldPrices/Latest";
  const res = UrlFetchApp.fetch(url, {
    method: "get",
    muteHttpExceptions: true,
    headers: {
      "Accept": "application/json",
      "User-Agent": "Mozilla/5.0"
    }
  });

  const code = res.getResponseCode();
  const body = res.getContentText();

  if (code !== 200) {
    throw new Error("GoldTraders HTTP " + code + ": " + body.slice(0, 300));
  }

  const data = JSON.parse(body);
  validateGoldTraders(data);
  return data;
}

function validateGoldTraders(data) {
  if (!data) throw new Error("Empty response");
  if (!data.goldPriceID) throw new Error("Missing goldPriceID");
  if (data.bL_BuyPrice == null) throw new Error("Missing bL_BuyPrice");
  if (data.bL_SellPrice == null) throw new Error("Missing bL_SellPrice");
  if (data.bahtPerUSD == null) throw new Error("Missing bahtPerUSD");
  return true;
}

// ==========================================
// Web App Endpoints (doGet)
// ==========================================

function doGet(e) {
  const callback = e && e.parameter && e.parameter.callback;
  try {
    let cached = null;
    try {
      cached = CacheService.getScriptCache().get(LATEST_GOLD_CACHE_KEY);
    } catch (e) {
      Logger.log("Could not read price cache: " + e.toString());
    }
    const data = cached ? JSON.parse(cached) : getLatestGoldData_();
    if (!cached) {
      try {
        CacheService.getScriptCache().put(LATEST_GOLD_CACHE_KEY, JSON.stringify(data), 60);
      } catch (e) {
        Logger.log("Could not store price cache: " + e.toString());
      }
    }
    data.generatedAt = formatThaiTime_(new Date(), true);

    return goldResponse_({
      success: true,
      data: data
    }, callback);
  } catch (error) {
    return goldResponse_({
      success: false,
      error: error.message
    }, callback);
  }
}

function goldResponse_(payload, callback) {
  // Fixed callback only: public price data, no arbitrary JavaScript injection.
  if (callback === "__alongkornGoldCallback") {
    return ContentService
      .createTextOutput("__alongkornGoldCallback(" + JSON.stringify(payload) + ");")
      .setMimeType(ContentService.MimeType.JAVASCRIPT);
  }
  return jsonResponse_(payload);
}

function getLatestGoldData_() {
  const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
  const sheet = ss.getSheetByName("PriceLog");

  if (!sheet || sheet.getLastRow() < 2) {
    throw new Error("ไม่พบข้อมูลราคาใน PriceLog");
  }

  const lastRow = sheet.getLastRow();
  const headers = getCleanHeaders_(sheet);
  const values = sheet.getRange(lastRow, 1, 1, headers.length).getValues()[0];

  const getValue = (header, fallback = null) => {
    const index = headers.indexOf(header);
    return index >= 0 ? values[index] : fallback;
  };

  const recordedAt = getValue("บันทึกเมื่อ");
  const asTime = getValue("เวลาประกาศ");

  return {
    source: "goldtraders_sheet",
    generatedAt: formatThaiTime_(new Date(), true),
    recordedAt: formatThaiTime_(recordedAt, true),
    goldPriceID: Number(getValue("GoldPriceID")),
    asTime: formatThaiTime_(asTime),
    seq: Number(getValue("ครั้งที่")),
    goldBar: {
      buy: Number(getValue("สมาคมซื้อ", 0)),
      sell: Number(getValue("สมาคมขาย", 0))
    },
    exchange: {
      usdThb: Number(getValue("USD/THB", 0))
    },
    change: {
      fromPrevRound: Number(getValue("เทียบรอบก่อน", 0)),
      fromYesterday: Number(getValue("เทียบเมื่อวาน", 0))
    }
  };
}

function jsonResponse_(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
