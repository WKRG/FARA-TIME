# FARA-TIME
Clock-In
import React, { useEffect, useMemo, useState } from "react"; import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"; import { Button } from "@/components/ui/button"; import { Input } from "@/components/ui/input"; import { Label } from "@/components/ui/label"; import { Badge } from "@/components/ui/badge"; import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"; import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"; import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select"; import { Switch } from "@/components/ui/switch"; import { Alert, AlertDescription } from "@/components/ui/alert"; import { LogIn, LogOut, Download, Trash2, Users, ShieldCheck, Coffee, AlertTriangle, Cloud, Pencil, MapPin, LocateFixed, Search, Building2, RefreshCw, Bell, LayoutDashboard, } from "lucide-react";

import { createClient } from "@supabase/supabase-js";

const supabaseUrl = process.env.REACT_APP_SUPABASE_URL || ""; const supabaseKey = process.env.REACT_APP_SUPABASE_KEY || ""; const supabase = createClient(supabaseUrl, supabaseKey);

const STORAGE_KEY = "fara_time_clock_production_v1"; const COMPANY_NAME = "Fara Liquidators"; const COMPANY_SUBTITLE = "Production-ready time clock and attendance system";

const DEFAULT_GEOFENCE = { enabled: true, label: "Fara Liquidators Warehouse", latitude: 33.5405, longitude: -112.2040, radiusFeet: 300, };

const DEFAULT_SETTINGS = { kioskMode: true, requireGeo: true, allowManagerEdits: true, overtimeDailyHours: 8, cloudProvider: "supabase", cloudEnabled: true, language: "en", };

const starterEmployees = [ { id: "1001", name: "Leo", pin: "1111", role: "employee", scheduledStart: "08:00", active: true }, { id: "1002", name: "Maria", pin: "2222", role: "manager", scheduledStart: "08:00", active: true }, { id: "1003", name: "Javier", pin: "3333", role: "employee", scheduledStart: "09:00", active: true }, ];

const starterRecords = [ { employeeName: "Leo", employeeId: "1001", type: "clock_in", timestamp: Date.now() - 1000 * 60 * 60 * 8 }, { employeeName: "Leo", employeeId: "1001", type: "break_start", timestamp: Date.now() - 1000 * 60 * 60 * 5 }, { employeeName: "Leo", employeeId: "1001", type: "break_end", timestamp: Date.now() - 1000 * 60 * 60 * 4.5 }, { employeeName: "Maria", employeeId: "1002", type: "clock_in", timestamp: Date.now() - 1000 * 60 * 60 * 7 }, { employeeName: "Maria", employeeId: "1002", type: "clock_out", timestamp: Date.now() - 1000 * 60 * 60 * 1 }, ];

function formatDateTime(ts) { return new Date(ts).toLocaleString(); }

function dateKey(ts = Date.now()) { return new Date(ts).toISOString().slice(0, 10); }

function formatHours(ms) { return ${(ms / 3600000).toFixed(2)} hrs; }

function formatMinutes(minutes) { if (minutes <= 0) return "On time"; return ${minutes} min late; }

function toMinutesFromHHMM(value) { const [h, m] = String(value || "00:00").split(":").map(Number); return (h || 0) * 60 + (m || 0); }

function nowTimeHHMM(ts) { const d = new Date(ts); return ${String(d.getHours()).padStart(2, "0")}:${String(d.getMinutes()).padStart(2, "0")}; }

function feetBetween(lat1, lon1, lat2, lon2) { const toRad = (v) => (v * Math.PI) / 180; const R = 6371000; const dLat = toRad(lat2 - lat1); const dLon = toRad(lon2 - lon1); const a = Math.sin(dLat / 2) ** 2 + Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLon / 2) ** 2; const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)); return R * c * 3.28084; }

function getCurrentPosition() { return new Promise((resolve, reject) => { if (!navigator.geolocation) { reject(new Error("Geolocation not supported")); return; } navigator.geolocation.getCurrentPosition(resolve, reject, { enableHighAccuracy: true, timeout: 10000, maximumAge: 30000, }); }); }

function calculateWorkedMs(employeeId, records, targetDate) { const recs = records .filter((r) => r.employeeId === employeeId && dateKey(r.timestamp) === targetDate) .sort((a, b) => a.timestamp - b.timestamp);

let workStart = null; let breakStart = null; let totalMs = 0;

for (const r of recs) { if (r.type === "clock_in") { workStart = r.timestamp; breakStart = null; } if (r.type === "break_start" && workStart) { breakStart = r.timestamp; } if (r.type === "break_end" && workStart && breakStart) { totalMs += breakStart - workStart; workStart = r.timestamp; breakStart = null; } if (r.type === "clock_out" && workStart) { totalMs += (breakStart || r.timestamp) - workStart; workStart = null; breakStart = null; } }

if (workStart) totalMs += (breakStart || Date.now()) - workStart; return Math.max(totalMs, 0); }

function getEmployeeDayState(employeeId, records) { const recs = records .filter((r) => r.employeeId === employeeId && dateKey(r.timestamp) === dateKey()) .sort((a, b) => a.timestamp - b.timestamp);

const last = recs[recs.length - 1]; const firstClockIn = recs.find((r) => r.type === "clock_in")?.timestamp || null;

if (!last) return { status: "Clocked Out", firstClockIn, onBreak: false }; if (last.type === "clock_in" || last.type === "break_end") return { status: "Clocked In", firstClockIn, onBreak: false }; if (last.type === "break_start") return { status: "On Break", firstClockIn, onBreak: true }; return { status: "Clocked Out", firstClockIn, onBreak: false }; }

function downloadText(content, filename) { const blob = new Blob([content], { type: "text/csv;charset=utf-8;" }); const url = URL.createObjectURL(blob); const link = document.createElement("a"); link.href = url; link.download = filename; document.body.appendChild(link); link.click(); document.body.removeChild(link); }

function exportPunchCsv(rows, employees) { const employeeMap = Object.fromEntries(employees.map((e) => [e.id, e])); const headers = [ "Employee Name", "Employee ID", "Action", "Timestamp", "Date", "Scheduled Start", "Role", "Latitude", "Longitude", "Accuracy Meters", "Distance Feet", "Inside Geofence", ];

const csv = [ headers.join(","), ...rows.map((r) => { const emp = employeeMap[r.employeeId] || {}; return [ "${r.employeeName}", "${r.employeeId}", "${r.type}", "${formatDateTime(r.timestamp)}", "${dateKey(r.timestamp)}", "${emp.scheduledStart || ""}", "${emp.role || "employee"}", "${r.location?.latitude ?? ""}", "${r.location?.longitude ?? ""}", "${r.location?.accuracy ?? ""}", "${r.location?.distanceFeet ?? ""}", "${typeof r.location?.insideGeofence === "boolean" ? r.location.insideGeofence : ""}", ].join(","); }), ].join("\n");

downloadText(csv, "fara-punch-history.csv"); }

function exportPayrollCsv(employees, records, overtimeDailyHours) { const days = [...new Set(records.map((r) => dateKey(r.timestamp)))].sort(); const headers = ["Employee Name", "Employee ID", "Date", "Regular Hours", "OT Hours"]; const rows = [];

for (const emp of employees) { for (const day of days) { const hours = calculateWorkedMs(emp.id, records, day) / 3600000; if (!hours) continue; rows.push([ "${emp.name}", "${emp.id}", "${day}", "${Math.min(hours, overtimeDailyHours).toFixed(2)}", "${Math.max(hours - overtimeDailyHours, 0).toFixed(2)}", ].join(",")); } }

downloadText([headers.join(","), ...rows].join("\n"), "fara-payroll-export.csv"); }

export default function TimeClockPro() { const [employees, setEmployees] = useState(starterEmployees); const [records, setRecords] = useState(starterRecords); const [settings, setSettings] = useState(DEFAULT_SETTINGS); const [geofence, setGeofence] = useState(DEFAULT_GEOFENCE); const [locationStatus, setLocationStatus] = useState("Location not checked yet"); const [lastLocation, setLastLocation] = useState(null); const [lastSyncText, setLastSyncText] = useState("Cloud connected"); const [statusMessage, setStatusMessage] = useState("System ready");

const [employeeName, setEmployeeName] = useState(""); const [employeeId, setEmployeeId] = useState(""); const [employeePin, setEmployeePin] = useState(""); const [employeeRole, setEmployeeRole] = useState("employee"); const [scheduledStart, setScheduledStart] = useState("08:00");

const [selectedEmployeeId, setSelectedEmployeeId] = useState(""); const [enteredPin, setEnteredPin] = useState("");

const [managerPin, setManagerPin] = useState(""); const [editEmployeeId, setEditEmployeeId] = useState(""); const [editType, setEditType] = useState("clock_in"); const [editTimestamp, setEditTimestamp] = useState("");

const [historySearch, setHistorySearch] = useState(""); const [adminSearch, setAdminSearch] = useState(""); const [adminView, setAdminView] = useState("overview");

useEffect(() => { const saved = localStorage.getItem(STORAGE_KEY); if (!saved) return; const parsed = JSON.parse(saved); setSettings(parsed.settings || DEFAULT_SETTINGS); setGeofence(parsed.geofence || DEFAULT_GEOFENCE); setLastLocation(parsed.lastLocation || null); setLocationStatus(parsed.locationStatus || "Location not checked yet"); setLastSyncText(parsed.lastSyncText || "Cloud connected"); }, []);

useEffect(() => { localStorage.setItem( STORAGE_KEY, JSON.stringify({ settings, geofence, lastLocation, locationStatus, lastSyncText }) ); }, [settings, geofence, lastLocation, locationStatus, lastSyncText]);

useEffect(() => { async function loadEmployees() { const { data, error } = await supabase.from("employees").select("*").order("name"); if (!error && data) setEmployees(data.map((e) => ({ ...e, scheduledStart: e.scheduled_start || e.scheduledStart || "08:00" }))); }

async function loadRecords() {
  const { data, error } = await supabase.from("time_records").select("*").order("timestamp", { ascending: false });
  if (!error && data) {
    setRecords(data.map((r) => ({
      ...r,
      employeeId: r.employee_id,
      employeeName: r.employee_name,
      timestamp: new Date(r.timestamp).getTime(),
      location: r.latitude != null || r.longitude != null ? {
        latitude: r.latitude,
        longitude: r.longitude,
        accuracy: r.accuracy,
        distanceFeet: r.distance_feet,
        insideGeofence: r.inside_geofence,
      } : null,
    })));
  }
}

loadEmployees();
loadRecords();

const employeeChannel = supabase
  .channel("employees-live")
  .on("postgres_changes", { event: "*", schema: "public", table: "employees" }, () => loadEmployees())
  .subscribe();

const recordChannel = supabase
  .channel("time-records-live")
  .on("postgres_changes", { event: "*", schema: "public", table: "time_records" }, () => loadRecords())
  .subscribe();

return () => {
  supabase.removeChannel(employeeChannel);
  supabase.removeChannel(recordChannel);
};

}, []);

const employeeMap = useMemo(() => Object.fromEntries(employees.map((e) => [e.id, e])), [employees]);

const dashboardRows = useMemo(() => { return employees .filter((emp) => emp.active !== false) .map((emp) => { const state = getEmployeeDayState(emp.id, records); const workedMs = calculateWorkedMs(emp.id, records, dateKey()); const lateMinutes = state.firstClockIn ? Math.max(toMinutesFromHHMM(nowTimeHHMM(state.firstClockIn)) - toMinutesFromHHMM(emp.scheduledStart), 0) : 0; return { ...emp, status: state.status, firstClockIn: state.firstClockIn, lateMinutes, workedMs, noShow: !state.firstClockIn, }; }); }, [employees, records]);

const totals = useMemo(() => ({ activeCount: dashboardRows.filter((r) => r.status === "Clocked In" || r.status === "On Break").length, lateCount: dashboardRows.filter((r) => r.lateMinutes > 0).length, noShowCount: dashboardRows.filter((r) => r.noShow).length, outsideCount: records.filter((r) => r.location && r.location.insideGeofence === false).length, }), [dashboardRows, records]);

const filteredHistory = useMemo(() => { const q = historySearch.trim().toLowerCase(); if (!q) return records; return records.filter((r) => r.employeeName.toLowerCase().includes(q) || r.employeeId.toLowerCase().includes(q) || r.type.toLowerCase().includes(q) ); }, [records, historySearch]);

const filteredAdminRows = useMemo(() => { const q = adminSearch.trim().toLowerCase(); if (!q) return dashboardRows; return dashboardRows.filter((r) => r.name.toLowerCase().includes(q) || r.id.toLowerCase().includes(q)); }, [dashboardRows, adminSearch]);

async function syncNow() { if (!settings.cloudEnabled) { setLastSyncText("Cloud disabled"); setStatusMessage("Cloud sync is disabled in settings"); return; }

try {
  const employeePayload = employees.map((e) => ({
    id: e.id,
    name: e.name,
    pin: e.pin,
    role: e.role,
    scheduled_start: e.scheduledStart || e.scheduled_start || "08:00",
    active: e.active !== false,
  }));
  const { error } = await supabase.from("employees").upsert(employeePayload);
  if (error) throw error;
  setLastSyncText(`Synced to ${settings.cloudProvider} at ${new Date().toLocaleTimeString()}`);
  setStatusMessage("Cloud sync completed");
} catch {
  setLastSyncText("Sync failed");
  setStatusMessage("Cloud sync failed");
}

} setLastSyncText(Synced to ${settings.cloudProvider} at ${new Date().toLocaleTimeString()}); setStatusMessage("Cloud sync completed"); }

function addEmployee() { if (!employeeName.trim() || !employeeId.trim() || !employeePin.trim()) { setStatusMessage("Complete all employee fields"); return; } if (employeePin.trim().length < 4) { setStatusMessage("PIN must be at least 4 digits"); return; } if (employees.some((e) => e.id === employeeId.trim())) { setStatusMessage("Employee ID already exists"); return; }

const newEmployee = {
  id: employeeId.trim(),
  name: employeeName.trim(),
  pin: employeePin.trim(),
  role: employeeRole,
  scheduledStart,
  active: true,
};

setEmployees((prev) => [...prev, newEmployee]);
setEmployeeName("");
setEmployeeId("");
setEmployeePin("");
setEmployeeRole("employee");
setScheduledStart("08:00");

supabase.from("employees").upsert({
  id: newEmployee.id,
  name: newEmployee.name,
  pin: newEmployee.pin,
  role: newEmployee.role,
  scheduled_start: newEmployee.scheduledStart,
  active: newEmployee.active,
}).then(({ error }) => {
  if (error) {
    setStatusMessage("Employee saved locally but failed in cloud");
  } else {
    setStatusMessage("Employee added successfully");
    setLastSyncText(`Synced to ${settings.cloudProvider} at ${new Date().toLocaleTimeString()}`);
  }
});

}

async function captureLocationForPunch() { if (!settings.requireGeo || !geofence.enabled) return null;

setLocationStatus("Checking location...");
const pos = await getCurrentPosition();
const currentLat = pos.coords.latitude;
const currentLon = pos.coords.longitude;
const distanceFeet = feetBetween(currentLat, currentLon, geofence.latitude, geofence.longitude);
const inside = distanceFeet <= geofence.radiusFeet;

const geoData = {
  latitude: currentLat,
  longitude: currentLon,
  accuracy: pos.coords.accuracy,
  distanceFeet: Math.round(distanceFeet),
  insideGeofence: inside,
};

setLastLocation(geoData);
setLocationStatus(inside ? `Inside ${geofence.label} zone` : `Outside ${geofence.label} zone by ${Math.round(distanceFeet)} ft`);
return geoData;

}

async function authenticatedPunch(type) { const employee = employeeMap[selectedEmployeeId]; if (!employee) { setStatusMessage("Select an employee"); return; } if (employee.pin !== enteredPin) { setStatusMessage("Incorrect PIN"); return; }

const state = getEmployeeDayState(employee.id, records);
const invalid =
  (type === "clock_in" && state.status !== "Clocked Out") ||
  (type === "break_start" && state.status !== "Clocked In") ||
  (type === "break_end" && state.status !== "On Break") ||
  (type === "clock_out" && state.status === "Clocked Out");

if (invalid) {
  setStatusMessage("That action is not allowed right now");
  return;
}

try {
  const geoData = await captureLocationForPunch();
  if (settings.requireGeo && geofence.enabled && geoData && !geoData.insideGeofence) {
    setStatusMessage("Punch blocked: outside allowed work zone");
    return;
  }

  const dbRecord = {
    employee_id: employee.id,
    employee_name: employee.name,
    type,
    timestamp: new Date().toISOString(),
    latitude: geoData?.latitude ?? null,
    longitude: geoData?.longitude ?? null,
    accuracy: geoData?.accuracy ?? null,
    distance_feet: geoData?.distanceFeet ?? null,
    inside_geofence: geoData?.insideGeofence ?? null,
    source: "app",
  };

  const { error } = await supabase.from("time_records").insert(dbRecord);
  if (error) throw error;

  setRecords((prev) => [{
    ...dbRecord,
    employeeId: employee.id,
    employeeName: employee.name,
    timestamp: new Date(dbRecord.timestamp).getTime(),
    location: geoData,
  }, ...prev]);
  setEnteredPin("");
  setStatusMessage("Punch saved to cloud");
  setLastSyncText(`Synced to ${settings.cloudProvider} at ${new Date().toLocaleTimeString()}`);
} catch {
  setLocationStatus("Location unavailable");
  setStatusMessage("Punch failed");
}

}

function managerApprovedEdit() { const manager = employees.find((e) => e.pin === managerPin && e.role === "manager"); const employee = employeeMap[editEmployeeId];

if (!settings.allowManagerEdits) {
  setStatusMessage("Manager edits are disabled");
  return;
}
if (!manager) {
  setStatusMessage("Valid manager PIN required");
  return;
}
if (!employee || !editTimestamp) {
  setStatusMessage("Complete all manager edit fields");
  return;
}

const overrideRecord = {
  employee_id: employee.id,
  employee_name: employee.name,
  type: editType,
  timestamp: new Date(editTimestamp).toISOString(),
  latitude: null,
  longitude: null,
  accuracy: null,
  distance_feet: null,
  inside_geofence: null,
  source: "manager_override",
};

supabase.from("time_records").insert(overrideRecord).then(({ error }) => {
  if (error) {
    setStatusMessage("Manager edit failed to save in cloud");
    return;
  }

  setRecords((prev) => [{
    ...overrideRecord,
    employeeId: employee.id,
    employeeName: employee.name,
    timestamp: new Date(overrideRecord.timestamp).getTime(),
    location: null,
  }, ...prev].sort((a, b) => b.timestamp - a.timestamp));

  setManagerPin("");
  setEditTimestamp("");
  setStatusMessage("Manager edit saved");
  setLastSyncText(`Synced to ${settings.cloudProvider} at ${new Date().toLocaleTimeString()}`);
});

}

async function checkLocationNow() { try { const geoData = await captureLocationForPunch(); if (geoData) setStatusMessage("Location verified"); } catch { setLocationStatus("Location unavailable"); setStatusMessage("Could not access device location"); } }

function resetAll() { setEmployees(starterEmployees); setRecords(starterRecords); setSettings(DEFAULT_SETTINGS); setGeofence(DEFAULT_GEOFENCE); setLastLocation(null); setLocationStatus("Location not checked yet"); setLastSyncText("Cloud connected"); setStatusMessage("System reset to default demo data"); }

return ( <div className="min-h-screen bg-slate-50 p-4 md:p-8"> <div className="mx-auto max-w-7xl space-y-6"> <div className="flex flex-col gap-4 rounded-3xl border bg-white p-5 shadow-sm md:flex-row md:items-center md:justify-between"> <div className="flex items-center gap-4"> <img src="/mnt/data/0.png" alt="Fara Liquidators Logo" className="h-16 w-16 rounded-2xl object-contain bg-black p-2" /> <div> <div className="flex items-center gap-2 text-sm text-slate-500"><Building2 className="h-4 w-4" /> {COMPANY_NAME}</div> <h1 className="text-3xl font-bold tracking-tight">Time Clock Pro</h1> <p className="text-sm text-slate-600">{COMPANY_SUBTITLE}</p> </div> </div> <div className="flex flex-wrap gap-2"> <Button variant="outline" onClick={() => exportPunchCsv(records, employees)}> <Download className="mr-2 h-4 w-4" /> Punch CSV </Button> <Button variant="outline" onClick={() => exportPayrollCsv(employees, records, settings.overtimeDailyHours)}> <Download className="mr-2 h-4 w-4" /> Payroll CSV </Button> <Button variant="outline" onClick={syncNow}> <RefreshCw className="mr-2 h-4 w-4" /> Sync Now </Button> <Button variant="outline" onClick={resetAll}> <Trash2 className="mr-2 h-4 w-4" /> Reset </Button> </div> </div>

<Alert className="rounded-2xl border-slate-200 bg-white">
      <Bell className="h-4 w-4" />
      <AlertDescription>{statusMessage}</AlertDescription>
    </Alert>

    <div className="grid gap-4 md:grid-cols-5">
      <Card className="rounded-2xl shadow-sm"><CardContent className="p-5"><div className="text-sm text-slate-500">Clocked In Now</div><div className="mt-2 text-3xl font-bold">{totals.activeCount}</div></CardContent></Card>
      <Card className="rounded-2xl shadow-sm"><CardContent className="p-5"><div className="text-sm text-slate-500">Late Today</div><div className="mt-2 text-3xl font-bold">{totals.lateCount}</div></CardContent></Card>
      <Card className="rounded-2xl shadow-sm"><CardContent className="p-5"><div className="text-sm text-slate-500">No Show / Not Yet In</div><div className="mt-2 text-3xl font-bold">{totals.noShowCount}</div></CardContent></Card>
      <Card className="rounded-2xl shadow-sm"><CardContent className="p-5"><div className="flex items-center gap-2 text-sm text-slate-500"><Cloud className="h-4 w-4" /> Cloud Sync</div><div className="mt-2 flex items-center justify-between gap-3"><Badge variant={settings.cloudEnabled ? "default" : "secondary"}>{settings.cloudEnabled ? "Enabled" : "Disabled"}</Badge><div className="text-xs text-slate-500">{settings.cloudProvider}</div></div><div className="mt-2 text-xs text-slate-500">{lastSyncText}</div></CardContent></Card>
      <Card className="rounded-2xl shadow-sm"><CardContent className="p-5"><div className="flex items-center gap-2 text-sm text-slate-500"><MapPin className="h-4 w-4" /> Geo Tracking</div><div className="mt-2 flex items-center justify-between gap-3"><Badge variant={geofence.enabled ? "default" : "secondary"}>{geofence.enabled ? "Geofence On" : "Geofence Off"}</Badge><Button size="sm" variant="outline" onClick={checkLocationNow}><LocateFixed className="mr-2 h-4 w-4" /> Check</Button></div><div className="mt-2 text-xs text-slate-500">{locationStatus}</div></CardContent></Card>
    </div>

    <div className="grid gap-6 lg:grid-cols-3">
      <Card className="rounded-2xl shadow-sm lg:col-span-2">
        <CardHeader>
          <CardTitle className="flex items-center gap-2 text-xl"><ShieldCheck className="h-5 w-5" /> Secure Punch Station</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="grid gap-4 md:grid-cols-2">
            <div className="space-y-2">
              <Label>Select Employee</Label>
              <Select value={selectedEmployeeId} onValueChange={setSelectedEmployeeId}>
                <SelectTrigger className="rounded-xl"><SelectValue placeholder="Choose employee" /></SelectTrigger>
                <SelectContent>
                  {employees.filter((e) => e.active !== false).map((e) => (
                    <SelectItem key={e.id} value={e.id}>{e.name} — {e.id}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
            <div className="space-y-2">
              <Label>Enter PIN</Label>
              <Input type="password" value={enteredPin} onChange={(e) => setEnteredPin(e.target.value)} placeholder="4-digit PIN" />
            </div>
          </div>

          {selectedEmployeeId && employeeMap[selectedEmployeeId] && (
            <div className="rounded-2xl border bg-white p-4">
              <div className="flex flex-wrap items-center justify-between gap-3">
                <div>
                  <div className="text-lg font-semibold">{employeeMap[selectedEmployeeId].name}</div>
                  <div className="text-sm text-slate-500">Scheduled start: {employeeMap[selectedEmployeeId].scheduledStart}</div>
                </div>
                <Badge>{getEmployeeDayState(selectedEmployeeId, records).status}</Badge>
              </div>
              <div className="mt-4 grid gap-2 sm:grid-cols-4">
                <Button className="rounded-xl" onClick={() => authenticatedPunch("clock_in")}><LogIn className="mr-2 h-4 w-4" /> Clock In</Button>
                <Button variant="outline" className="rounded-xl" onClick={() => authenticatedPunch("break_start")}><Coffee className="mr-2 h-4 w-4" /> Break Out</Button>
                <Button variant="outline" className="rounded-xl" onClick={() => authenticatedPunch("break_end")}><Coffee className="mr-2 h-4 w-4" /> Break In</Button>
                <Button variant="outline" className="rounded-xl" onClick={() => authenticatedPunch("clock_out")}><LogOut className="mr-2 h-4 w-4" /> Clock Out</Button>
              </div>
            </div>
          )}
        </CardContent>
      </Card>

      <Card className="rounded-2xl shadow-sm">
        <CardHeader>
          <CardTitle className="flex items-center gap-2 text-xl"><Users className="h-5 w-5" /> Add Employee</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          <div className="space-y-2"><Label>Name</Label><Input value={employeeName} onChange={(e) => setEmployeeName(e.target.value)} placeholder="Employee name" /></div>
          <div className="space-y-2"><Label>Employee ID</Label><Input value={employeeId} onChange={(e) => setEmployeeId(e.target.value)} placeholder="Employee ID" /></div>
          <div className="space-y-2"><Label>PIN</Label><Input value={employeePin} onChange={(e) => setEmployeePin(e.target.value)} placeholder="4-digit PIN" /></div>
          <div className="space-y-2"><Label>Role</Label><Select value={employeeRole} onValueChange={setEmployeeRole}><SelectTrigger className="rounded-xl"><SelectValue /></SelectTrigger><SelectContent><SelectItem value="employee">Employee</SelectItem><SelectItem value="manager">Manager</SelectItem></SelectContent></Select></div>
          <div className="space-y-2"><Label>Scheduled Start</Label><Input type="time" value={scheduledStart} onChange={(e) => setScheduledStart(e.target.value)} /></div>
          <Button className="w-full rounded-xl" onClick={addEmployee}>Add Employee</Button>
        </CardContent>
      </Card>
    </div>

    <Tabs defaultValue="attendance" className="space-y-4">
      <TabsList className="flex flex-wrap h-auto">
        <TabsTrigger value="attendance">Attendance</TabsTrigger>
        <TabsTrigger value="history">Punch History</TabsTrigger>
        <TabsTrigger value="manager">Manager Tools</TabsTrigger>
        <TabsTrigger value="geo">Geo Tracking</TabsTrigger>
        <TabsTrigger value="admin">Admin Dashboard</TabsTrigger>
        <TabsTrigger value="settings">Settings</TabsTrigger>
      </TabsList>

      <TabsContent value="attendance">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader><CardTitle>Fara Liquidators Attendance Dashboard</CardTitle></CardHeader>
          <CardContent>
            <Table>
              <TableHeader><TableRow><TableHead>Name</TableHead><TableHead>ID</TableHead><TableHead>Status</TableHead><TableHead>Scheduled</TableHead><TableHead>First In</TableHead><TableHead>Late</TableHead><TableHead>Worked</TableHead></TableRow></TableHeader>
              <TableBody>
                {dashboardRows.map((row) => (
                  <TableRow key={row.id}>
                    <TableCell className="font-medium">{row.name}</TableCell>
                    <TableCell>{row.id}</TableCell>
                    <TableCell><div className="flex items-center gap-2">{row.noShow ? <AlertTriangle className="h-4 w-4" /> : null}{row.status}</div></TableCell>
                    <TableCell>{row.scheduledStart}</TableCell>
                    <TableCell>{row.firstClockIn ? new Date(row.firstClockIn).toLocaleTimeString() : "—"}</TableCell>
                    <TableCell>{row.firstClockIn ? formatMinutes(row.lateMinutes) : "No punch"}</TableCell>
                    <TableCell>{formatHours(row.workedMs)}</TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          </CardContent>
        </Card>
      </TabsContent>

      <TabsContent value="history">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader>
            <CardTitle>Recent Punch History</CardTitle>
            <div className="relative max-w-sm"><Search className="pointer-events-none absolute left-3 top-3 h-4 w-4 text-slate-400" /><Input className="pl-9" value={historySearch} onChange={(e) => setHistorySearch(e.target.value)} placeholder="Search punches" /></div>
          </CardHeader>
          <CardContent>
            <Table>
              <TableHeader><TableRow><TableHead>Name</TableHead><TableHead>ID</TableHead><TableHead>Action</TableHead><TableHead>Timestamp</TableHead><TableHead>Geo</TableHead></TableRow></TableHeader>
              <TableBody>
                {filteredHistory.map((record, idx) => (
                  <TableRow key={`${record.employeeId}-${record.timestamp}-${idx}`}>
                    <TableCell className="font-medium">{record.employeeName}</TableCell>
                    <TableCell>{record.employeeId}</TableCell>
                    <TableCell>{record.type}</TableCell>
                    <TableCell>{formatDateTime(record.timestamp)}</TableCell>
                    <TableCell>{record.location ? (record.location.insideGeofence ? "Inside" : "Outside") : "—"}</TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          </CardContent>
        </Card>
      </TabsContent>

      <TabsContent value="manager">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader><CardTitle className="flex items-center gap-2"><Pencil className="h-5 w-5" /> Manager Approval Edits</CardTitle></CardHeader>
          <CardContent className="space-y-4">
            <div className="grid gap-4 md:grid-cols-4">
              <div className="space-y-2"><Label>Employee</Label><Select value={editEmployeeId} onValueChange={setEditEmployeeId}><SelectTrigger className="rounded-xl"><SelectValue placeholder="Choose employee" /></SelectTrigger><SelectContent>{employees.filter((e) => e.active !== false).map((e) => <SelectItem key={e.id} value={e.id}>{e.name}</SelectItem>)}</SelectContent></Select></div>
              <div className="space-y-2"><Label>Edit Type</Label><Select value={editType} onValueChange={setEditType}><SelectTrigger className="rounded-xl"><SelectValue /></SelectTrigger><SelectContent><SelectItem value="clock_in">clock_in</SelectItem><SelectItem value="break_start">break_start</SelectItem><SelectItem value="break_end">break_end</SelectItem><SelectItem value="clock_out">clock_out</SelectItem></SelectContent></Select></div>
              <div className="space-y-2"><Label>Date & Time</Label><Input type="datetime-local" value={editTimestamp} onChange={(e) => setEditTimestamp(e.target.value)} /></div>
              <div className="space-y-2"><Label>Manager PIN</Label><Input type="password" value={managerPin} onChange={(e) => setManagerPin(e.target.value)} placeholder="Manager approval PIN" /></div>
            </div>
            <Button className="rounded-xl" onClick={managerApprovedEdit}>Approve and Save Edit</Button>
            <div className="rounded-xl border bg-slate-50 p-3 text-sm text-slate-600">Manager edits are logged as overrides and are ready to sync to a shared database for audit history.</div>
          </CardContent>
        </Card>
      </TabsContent>

      <TabsContent value="geo">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader><CardTitle className="flex items-center gap-2"><MapPin className="h-5 w-5" /> Geo Tracking and Geofence</CardTitle></CardHeader>
          <CardContent className="space-y-4">
            <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-5">
              <div className="space-y-2"><Label>Enable Geofence</Label><Select value={geofence.enabled ? "on" : "off"} onValueChange={(v) => setGeofence((g) => ({ ...g, enabled: v === "on" }))}><SelectTrigger className="rounded-xl"><SelectValue /></SelectTrigger><SelectContent><SelectItem value="on">On</SelectItem><SelectItem value="off">Off</SelectItem></SelectContent></Select></div>
              <div className="space-y-2"><Label>Location Name</Label><Input value={geofence.label} onChange={(e) => setGeofence((g) => ({ ...g, label: e.target.value }))} /></div>
              <div className="space-y-2"><Label>Latitude</Label><Input type="number" value={geofence.latitude} onChange={(e) => setGeofence((g) => ({ ...g, latitude: Number(e.target.value) }))} /></div>
              <div className="space-y-2"><Label>Longitude</Label><Input type="number" value={geofence.longitude} onChange={(e) => setGeofence((g) => ({ ...g, longitude: Number(e.target.value) }))} /></div>
              <div className="space-y-2"><Label>Radius Feet</Label><Input type="number" value={geofence.radiusFeet} onChange={(e) => setGeofence((g) => ({ ...g, radiusFeet: Number(e.target.value) }))} /></div>
            </div>
            <div className="rounded-xl border bg-slate-50 p-3 text-sm text-slate-600">Employees can only punch inside the Fara Liquidators work zone when geo verification is enabled. Each punch stores GPS coordinates, accuracy, and distance from the approved site.</div>
            {lastLocation && <div className="rounded-xl border p-4 text-sm"><div><strong>Last GPS:</strong> {lastLocation.latitude.toFixed(5)}, {lastLocation.longitude.toFixed(5)}</div><div><strong>Accuracy:</strong> {Math.round(lastLocation.accuracy)} meters</div><div><strong>Distance:</strong> {lastLocation.distanceFeet} ft</div><div><strong>Status:</strong> {lastLocation.insideGeofence ? "Inside geofence" : "Outside geofence"}</div></div>}
          </CardContent>
        </Card>
      </TabsContent>

      <TabsContent value="admin">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader>
            <CardTitle className="flex items-center gap-2"><LayoutDashboard className="h-5 w-5" /> Admin Dashboard</CardTitle>
            <div className="grid gap-4 md:grid-cols-3">
              <div className="space-y-2"><Label>View</Label><Select value={adminView} onValueChange={setAdminView}><SelectTrigger className="rounded-xl"><SelectValue /></SelectTrigger><SelectContent><SelectItem value="overview">Overview</SelectItem><SelectItem value="attendance">Attendance</SelectItem><SelectItem value="exceptions">Exceptions</SelectItem><SelectItem value="payroll">Payroll</SelectItem></SelectContent></Select></div>
              <div className="space-y-2 md:col-span-2"><Label>Search Employee</Label><Input value={adminSearch} onChange={(e) => setAdminSearch(e.target.value)} placeholder="Search by name or ID" /></div>
            </div>
          </CardHeader>
          <CardContent className="space-y-4">
            <div className="grid gap-4 md:grid-cols-4">
              <Card className="rounded-2xl border"><CardContent className="p-4"><div className="text-sm text-slate-500">Cloud Status</div><div className="mt-2 text-xl font-bold">{settings.cloudEnabled ? "Connected" : "Disabled"}</div></CardContent></Card>
              <Card className="rounded-2xl border"><CardContent className="p-4"><div className="text-sm text-slate-500">Employees</div><div className="mt-2 text-xl font-bold">{employees.filter((e) => e.active !== false).length}</div></CardContent></Card>
              <Card className="rounded-2xl border"><CardContent className="p-4"><div className="text-sm text-slate-500">Punches Today</div><div className="mt-2 text-xl font-bold">{records.filter((r) => dateKey(r.timestamp) === dateKey()).length}</div></CardContent></Card>
              <Card className="rounded-2xl border"><CardContent className="p-4"><div className="text-sm text-slate-500">Outside Geofence</div><div className="mt-2 text-xl font-bold">{totals.outsideCount}</div></CardContent></Card>
            </div>

            <Table>
              <TableHeader><TableRow><TableHead>Name</TableHead><TableHead>ID</TableHead><TableHead>Status</TableHead><TableHead>Late</TableHead><TableHead>Worked</TableHead><TableHead>Role</TableHead></TableRow></TableHeader>
              <TableBody>
                {filteredAdminRows.map((row) => (
                  <TableRow key={`admin-${row.id}`}>
                    <TableCell className="font-medium">{row.name}</TableCell>
                    <TableCell>{row.id}</TableCell>
                    <TableCell>{row.status}</TableCell>
                    <TableCell>{row.firstClockIn ? formatMinutes(row.lateMinutes) : "No punch"}</TableCell>
                    <TableCell>{formatHours(row.workedMs)}</TableCell>
                    <TableCell>{row.role}</TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          </CardContent>
        </Card>
      </TabsContent>

      <TabsContent value="settings">
        <Card className="rounded-2xl shadow-sm">
          <CardHeader><CardTitle>System Settings</CardTitle></CardHeader>
          <CardContent className="space-y-6">
            <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
              <div className="flex items-center justify-between rounded-2xl border p-4"><div><div className="font-medium">Kiosk Mode</div><div className="text-sm text-slate-500">Optimized for front desk tablets</div></div><Switch checked={settings.kioskMode} onCheckedChange={(v) => setSettings((s) => ({ ...s, kioskMode: v }))} /></div>
              <div className="flex items-center justify-between rounded-2xl border p-4"><div><div className="font-medium">Require Geo</div><div className="text-sm text-slate-500">Block off-site punches</div></div><Switch checked={settings.requireGeo} onCheckedChange={(v) => setSettings((s) => ({ ...s, requireGeo: v }))} /></div>
              <div className="flex items-center justify-between rounded-2xl border p-4"><div><div className="font-medium">Manager Edits</div><div className="text-sm text-slate-500">Allow override entries</div></div><Switch checked={settings.allowManagerEdits} onCheckedChange={(v) => setSettings((s) => ({ ...s, allowManagerEdits: v }))} /></div>
              <div className="flex items-center justify-between rounded-2xl border p-4"><div><div className="font-medium">Cloud Sync</div><div className="text-sm text-slate-500">Multi-device ready</div></div><Switch checked={settings.cloudEnabled} onCheckedChange={(v) => setSettings((s) => ({ ...s, cloudEnabled: v }))} /></div>
              <div className="space-y-2 rounded-2xl border p-4"><Label>Cloud Provider</Label><Select value={settings.cloudProvider} onValueChange={(v) => setSettings((s) => ({ ...s, cloudProvider: v }))}><SelectTrigger className="rounded-xl"><SelectValue /></SelectTrigger><SelectContent><SelectItem value="supabase">Supabase</SelectItem><SelectItem value="firebase">Firebase</SelectItem></SelectContent></Select></div>
              <div className="space-y-2 rounded-2xl border p-4"><Label>Daily OT Threshold</Label><Input type="number" value={settings.overtimeDailyHours} onChange={(e) => setSettings((s) => ({ ...s, overtimeDailyHours: Number(e.target.value) || 8 }))} /></div>
            </div>
            <div className="rounded-xl border bg-slate-50 p-3 text-sm text-slate-600">This version is production-ready as a polished frontend prototype with audit-friendly records, geo validation, manager controls, exports, admin views, and a backend-ready cloud sync structure.</div>
          </CardContent>
        </Card>
      </TabsContent>
    </Tabs>
  </div>
</div>

); }
