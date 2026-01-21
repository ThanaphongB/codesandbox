import React, { useState, useMemo, useEffect, useRef } from "react";
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  Tooltip,
  ResponsiveContainer,
  PieChart,
  Pie,
  Cell,
} from "recharts";
import {
  LayoutDashboard,
  Database,
  Search,
  CheckCircle2,
  Zap,
  RefreshCw,
  MapPin,
  Menu,
  Sparkles,
  Sun,
  Moon,
  X,
  Filter,
  BrainCircuit,
  MessageSquare,
  Info,
  Hash,
  Timer,
  Activity,
  Layers,
  ChevronRight,
  FileText,
} from "lucide-react";

const DEFAULT_SHEET_URL =
  "https://docs.google.com/spreadsheets/d/e/2PACX-1vQIJ2F-lqM7hvMHO6koUTgSPxqHadraW9DR77K8EZ2c2w8mcwVAjgE_ASvggPefUfdm4rKzk0SsARPo/pub?gid=736553760&single=true&output=csv";
const apiKey = "";

const loadExternalScript = (id, src) => {
  return new Promise((resolve) => {
    if (document.getElementById(id)) {
      resolve();
      return;
    }
    const script = document.createElement("script");
    script.id = id;
    script.src = src;
    script.async = true;
    script.onload = () => resolve();
    document.head.appendChild(script);
  });
};

export default function App() {
  const [isDarkMode, setIsDarkMode] = useState(true);
  const [rawData, setRawData] = useState([]);
  const [loading, setLoading] = useState(false);
  const [searchTerm, setSearchTerm] = useState("");
  const [activeTab, setActiveTab] = useState("overview");
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const [filters, setFilters] = useState({ region: "All", status: "All" });
  const [selectedItem, setSelectedItem] = useState(null);
  const [isMapLibLoaded, setIsMapLibLoaded] = useState(false);

  const [aiAnalysis, setAiAnalysis] = useState("");
  const [analyzing, setAnalyzing] = useState(false);
  const [advising, setAdvising] = useState(false);
  const [siteAdvice, setSiteAdvice] = useState("");

  const mapRef = useRef(null);
  const mapInstance = useRef(null);
  const markersLayer = useRef(null);

  useEffect(() => {
    const init = async () => {
      await loadExternalScript(
        "papa",
        "https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.2/papaparse.min.js"
      );
      await loadExternalScript(
        "leaflet-js",
        "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
      );

      const link = document.createElement("link");
      link.rel = "stylesheet";
      link.href = "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";
      document.head.appendChild(link);

      setIsMapLibLoaded(true);
      fetchFromGoogleSheets(DEFAULT_SHEET_URL);
    };
    init();

    return () => {
      if (mapInstance.current) {
        mapInstance.current.remove();
        mapInstance.current = null;
      }
    };
  }, []);

  useEffect(() => {
    if (activeTab !== "overview" && mapInstance.current) {
      mapInstance.current.remove();
      mapInstance.current = null;
    }
  }, [activeTab]);

  const fetchFromGoogleSheets = (url) => {
    if (!window.Papa) return;
    setLoading(true);
    window.Papa.parse(url, {
      download: true,
      header: true,
      skipEmptyLines: true,
      complete: (results) => {
        const mapped = results.data.map((d, index) => ({
          id: index,
          ais1stTier: d["AIS 1st Tier"] || "Unnamed Node",
          siteCode: d["Site Code\n(MTS Site Alias)"] || d["Site Code"] || "N/A",
          jobId: d["Job ID"] || `TRK-${1000 + index}`,
          status: d["Status"] || "Active",
          region: d["Region"] || "Global",
          province: d["จังหวัด"] || d["Province"] || "N/A",
          action: d["Action"] || "No action specified",
          date:
            d["Date"] || d["วันที่"] || new Date().toLocaleDateString("th-TH"),
          details:
            d["Details"] || d["Remark"] || "No additional records available.",
          lat: d["Lat"] ? parseFloat(d["Lat"]) : null,
          lng: d["Long"] ? parseFloat(d["Long"]) : null,
        }));
        setRawData(mapped);
        setLoading(false);
      },
      error: () => setLoading(false),
    });
  };

  const filteredData = useMemo(() => {
    return rawData.filter((item) => {
      const matchRegion =
        filters.region === "All" || item.region === filters.region;
      const matchStatus =
        filters.status === "All" || item.status === filters.status;
      const matchSearch =
        searchTerm === "" ||
        [item.ais1stTier, item.jobId, item.siteCode, item.province].some(
          (val) => String(val).toLowerCase().includes(searchTerm.toLowerCase())
        );
      return matchRegion && matchStatus && matchSearch;
    });
  }, [rawData, filters, searchTerm]);

  useEffect(() => {
    if (
      activeTab === "overview" &&
      mapRef.current &&
      isMapLibLoaded &&
      window.L
    ) {
      if (!mapInstance.current) {
        mapInstance.current = window.L.map(mapRef.current, {
          zoomControl: false,
        }).setView([13.7367, 100.5231], 6);
        window.L.tileLayer(
          isDarkMode
            ? "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png"
            : "https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png",
          {
            attribution: "&copy; OpenStreetMap",
          }
        ).addTo(mapInstance.current);
        markersLayer.current = window.L.layerGroup().addTo(mapInstance.current);
      } else {
        mapInstance.current.eachLayer((layer) => {
          if (layer._url) {
            layer.setUrl(
              isDarkMode
                ? "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png"
                : "https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"
            );
          }
        });
      }

      if (markersLayer.current) {
        markersLayer.current.clearLayers();
        const pointsWithCoords = filteredData.filter((d) => d.lat && d.lng);
        pointsWithCoords.forEach((point) => {
          const marker = window.L.circleMarker([point.lat, point.lng], {
            radius: 8,
            fillColor: point.status.toLowerCase().includes("done")
              ? "#10b981"
              : "#f59e0b",
            color: "#fff",
            weight: 2,
            opacity: 1,
            fillOpacity: 0.8,
          });
          marker.bindPopup(`<b>${point.ais1stTier}</b><br/>${point.siteCode}`);
          marker.on("click", () => handleSelectItem(point));
          marker.addTo(markersLayer.current);
        });
        if (pointsWithCoords.length > 0) {
          mapInstance.current.fitBounds(
            window.L.latLngBounds(pointsWithCoords.map((p) => [p.lat, p.lng])),
            { padding: [50, 50] }
          );
        }
      }
    }
  }, [activeTab, filteredData, isDarkMode, isMapLibLoaded]);

  const stats = useMemo(() => {
    const total = filteredData.length;
    const done = filteredData.filter((d) =>
      d.status.toLowerCase().includes("done")
    ).length;
    const rate = total > 0 ? ((done / total) * 100).toFixed(1) : 0;
    return { total, done, pending: total - done, rate };
  }, [filteredData]);

  // Derived Data for Charts
  const chartData = useMemo(() => {
    const regionCounts = filteredData.reduce((acc, curr) => {
      acc[curr.region] = (acc[curr.region] || 0) + 1;
      return acc;
    }, {});
    return Object.entries(regionCounts).map(([name, count]) => ({
      name,
      count,
    }));
  }, [filteredData]);

  const pieData = useMemo(
    () => [
      { name: "Synced", value: stats.done },
      { name: "Queue", value: stats.pending },
    ],
    [stats]
  );

  const callGemini = async (prompt, systemPrompt) => {
    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            systemInstruction: { parts: [{ text: systemPrompt }] },
          }),
        }
      );
      const result = await response.json();
      return (
        result.candidates?.[0]?.content?.parts?.[0]?.text ||
        "ขออภัย ไม่มีการตอบกลับจาก AI"
      );
    } catch (e) {
      return "การเชื่อมต่อ AI ขัดข้อง กรุณาลองใหม่อีกครั้ง";
    }
  };

  const generateReport = async () => {
    setAnalyzing(true);
    const dataSnap = filteredData
      .slice(0, 10)
      .map((d) => `${d.ais1stTier}: ${d.status}`);
    const res = await callGemini(
      `Analyze: ${dataSnap.join(", ")}`,
      "Summarize network health in Thai. Be concise and professional."
    );
    setAiAnalysis(res);
    setAnalyzing(false);
  };

  const handleSelectItem = async (item) => {
    setSelectedItem(item);
    setAdvising(true);
    const res = await callGemini(
      `Analyze node: ${item.ais1stTier}, Action needed: ${item.action}`,
      "Provide 2 short bullet points in Thai as a network expert."
    );
    setSiteAdvice(res);
    setAdvising(false);
  };

  const theme = {
    bg: isDarkMode ? "bg-[#020617]" : "bg-[#f8fafc]",
    card: isDarkMode
      ? "bg-[#0f172a]/90 backdrop-blur-md border-[#1e293b]"
      : "bg-white border-[#e2e8f0] shadow-md",
    sidebar: isDarkMode ? "bg-[#020617]" : "bg-white shadow-2xl",
    text: isDarkMode ? "text-slate-100" : "text-slate-900",
    muted: isDarkMode ? "text-slate-400" : "text-slate-500",
    accent: "emerald-500",
  };

  const statusOptions = ["All", ...new Set(rawData.map((d) => d.status))];
  const regionOptions = ["All", ...new Set(rawData.map((d) => d.region))];

  return (
    <div
      className={`flex h-screen ${theme.bg} ${theme.text} overflow-hidden font-sans transition-colors duration-500 text-sm sm:text-base`}
    >
      <div
        className={`fixed inset-0 bg-black/70 z-[200] lg:hidden transition-opacity ${
          isSidebarOpen ? "opacity-100" : "opacity-0 pointer-events-none"
        }`}
        onClick={() => setIsSidebarOpen(false)}
      />

      <aside
        className={`fixed lg:static inset-y-0 left-0 w-72 xl:w-80 ${
          theme.sidebar
        } border-r ${
          isDarkMode ? "border-white/5" : "border-slate-200"
        } z-[210] transform transition-transform duration-300 lg:translate-x-0 ${
          isSidebarOpen ? "translate-x-0" : "-translate-x-full"
        } flex flex-col shrink-0`}
      >
        <div className="p-6 xl:p-8 overflow-y-auto custom-scrollbar">
          <div className="flex items-center gap-4 mb-10">
            <div className="p-2.5 bg-gradient-to-tr from-emerald-400 to-emerald-600 rounded-xl shadow-lg">
              <BrainCircuit className="text-white" size={24} />
            </div>
            <div>
              <h1 className="font-black text-2xl xl:text-3xl tracking-tighter leading-none">
                DTRS<span className="text-emerald-500">.AI</span>
              </h1>
              <p className="text-[10px] font-black text-slate-500 uppercase tracking-widest mt-1">
                Network Command
              </p>
            </div>
          </div>

          <nav className="space-y-1.5">
            <SideLink
              active={activeTab === "overview"}
              icon={LayoutDashboard}
              label="Dash Board"
              onClick={() => {
                setActiveTab("overview");
                setIsSidebarOpen(false);
              }}
              isDarkMode={isDarkMode}
            />
            <SideLink
              active={activeTab === "table"}
              icon={Layers}
              label="Data Matrix"
              onClick={() => {
                setActiveTab("table");
                setIsSidebarOpen(false);
              }}
              isDarkMode={isDarkMode}
            />
          </nav>

          <div className="mt-10 space-y-6">
            <h3 className="text-[10px] font-black text-slate-500 uppercase tracking-[0.2em] px-2 flex items-center gap-2">
              <Filter size={14} /> Filter Control
            </h3>
            <div className="space-y-4 px-2">
              <SelectBox
                label="Operational Region"
                value={filters.region}
                options={regionOptions}
                onChange={(v) => setFilters({ ...filters, region: v })}
                isDarkMode={isDarkMode}
              />
              <SelectBox
                label="Lifecycle Status"
                value={filters.status}
                options={statusOptions}
                onChange={(v) => setFilters({ ...filters, status: v })}
                isDarkMode={isDarkMode}
              />
            </div>
          </div>
        </div>

        <div className="mt-auto p-6 xl:p-8">
          <div
            className={`p-5 rounded-2xl ${
              isDarkMode ? "bg-white/5" : "bg-slate-100"
            } border border-white/5`}
          >
            <div className="flex items-center justify-between mb-2">
              <span className="text-[10px] font-black uppercase text-slate-500">
                Sync Capacity
              </span>
              <div className="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></div>
            </div>
            <div className="h-1.5 w-full bg-slate-800 rounded-full overflow-hidden">
              <div
                className="h-full bg-emerald-500 transition-all duration-1000"
                style={{ width: `${stats.rate}%` }}
              ></div>
            </div>
            <p className="mt-2 text-right text-lg font-black">{stats.rate}%</p>
          </div>
        </div>
      </aside>

      <main className="flex-1 flex flex-col relative overflow-hidden">
        <header
          className={`h-20 xl:h-24 border-b ${
            isDarkMode
              ? "border-white/5 bg-[#020617]/50"
              : "border-slate-200 bg-white/50"
          } backdrop-blur-xl flex items-center justify-between px-6 xl:px-12 z-[150] shrink-0`}
        >
          <div className="flex items-center gap-4">
            <button
              className="lg:hidden p-2.5 rounded-xl bg-slate-500/10"
              onClick={() => setIsSidebarOpen(true)}
            >
              <Menu size={24} />
            </button>
            <div className="relative group hidden sm:block">
              <Search
                className="absolute left-4 top-1/2 -translate-y-1/2 text-slate-500"
                size={18}
              />
              <input
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
                placeholder="Find node or province..."
                className={`w-48 xl:w-96 pl-11 pr-4 py-2.5 xl:py-3 rounded-xl font-bold outline-none border transition-all ${
                  isDarkMode
                    ? "bg-slate-900/50 border-white/10 focus:border-emerald-500/50 text-white"
                    : "bg-slate-100 border-slate-200 focus:border-emerald-500"
                }`}
              />
            </div>
          </div>

          <div className="flex items-center gap-3 xl:gap-4">
            <button
              onClick={() => setIsDarkMode(!isDarkMode)}
              className={`p-3 rounded-xl transition-all ${
                isDarkMode ? "bg-white/5" : "bg-slate-100"
              }`}
            >
              {isDarkMode ? (
                <Sun size={20} className="text-amber-400" />
              ) : (
                <Moon size={20} className="text-indigo-600" />
              )}
            </button>
            <div className="hidden md:flex flex-col text-right">
              <span className="text-[10px] font-black text-emerald-500 uppercase tracking-widest">
                Protocol
              </span>
              <span className="text-sm font-mono font-black">AI-OVR-2026</span>
            </div>
          </div>
        </header>

        <div className="flex-1 overflow-y-auto p-6 xl:p-12 custom-scrollbar">
          <div className="max-w-screen-2xl mx-auto space-y-8 xl:space-y-12 pb-16">
            {activeTab === "overview" && (
              <>
                <div className="grid grid-cols-2 lg:grid-cols-4 gap-4 xl:gap-8">
                  <MiniStat
                    label="Global Nodes"
                    value={stats.total}
                    icon={Database}
                    color="emerald"
                    isDarkMode={isDarkMode}
                  />
                  <MiniStat
                    label="Active Sync"
                    value={`${stats.rate}%`}
                    icon={Zap}
                    color="blue"
                    isDarkMode={isDarkMode}
                  />
                  <MiniStat
                    label="Queue"
                    value={stats.pending}
                    icon={Timer}
                    color="amber"
                    isDarkMode={isDarkMode}
                  />
                  <MiniStat
                    label="AI Status"
                    value="Online"
                    icon={Sparkles}
                    color="purple"
                    isDarkMode={isDarkMode}
                  />
                </div>

                <div
                  className={`${theme.card} rounded-[2rem] xl:rounded-[3rem] p-2 border overflow-hidden h-[400px] xl:h-[600px] relative`}
                >
                  <div
                    ref={mapRef}
                    className="w-full h-full rounded-[1.8rem] xl:rounded-[2.5rem] z-0"
                  />
                  <div className="absolute top-6 left-6 z-[10]">
                    <div
                      className={`px-4 py-2 rounded-xl backdrop-blur-md border border-white/10 flex items-center gap-2 ${
                        isDarkMode
                          ? "bg-black/50 text-white"
                          : "bg-white/80 text-black shadow-lg"
                      }`}
                    >
                      <div className="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></div>
                      <span className="text-[10px] font-black uppercase tracking-widest">
                        SITE OVER VIEW
                      </span>
                    </div>
                  </div>
                </div>

                {/* Grid สำหรับกราฟที่แก้ไขความสูงแล้ว */}
                <div className="grid grid-cols-1 lg:grid-cols-2 gap-8 xl:gap-12">
                  <div
                    className={`${theme.card} p-8 xl:p-12 rounded-[2.5rem] border`}
                  >
                    <h3 className="text-[11px] font-black text-slate-500 uppercase tracking-widest mb-10">
                      Regional Topology Distribution
                    </h3>
                    <div className="h-[300px] xl:h-[400px] w-full">
                      <ResponsiveContainer width="100%" height="100%">
                        <BarChart data={chartData}>
                          <XAxis
                            dataKey="name"
                            axisLine={false}
                            tickLine={false}
                            fontSize={10}
                            tick={{
                              fill: isDarkMode ? "#94a3b8" : "#64748b",
                              fontWeight: "bold",
                            }}
                          />
                          <Tooltip
                            cursor={{
                              fill: isDarkMode ? "#ffffff0a" : "#0000000a",
                            }}
                            contentStyle={{
                              backgroundColor: isDarkMode ? "#0f172a" : "#fff",
                              border: "none",
                              borderRadius: "16px",
                              fontSize: "12px",
                            }}
                          />
                          <Bar
                            dataKey="count"
                            fill="#10b981"
                            radius={[12, 12, 0, 0]}
                            barSize={40}
                          />
                        </BarChart>
                      </ResponsiveContainer>
                    </div>
                  </div>

                  <div
                    className={`${theme.card} p-8 xl:p-12 rounded-[2.5rem] border flex flex-col items-center justify-center`}
                  >
                    <div className="relative w-full h-[300px] xl:h-[400px]">
                      <ResponsiveContainer width="100%" height="100%">
                        <PieChart>
                          <Pie
                            data={pieData}
                            innerRadius="70%"
                            outerRadius="90%"
                            dataKey="value"
                            stroke="none"
                            paddingAngle={8}
                          >
                            <Cell fill="#10b981" />
                            <Cell fill={isDarkMode ? "#1e293b" : "#f1f5f9"} />
                          </Pie>
                        </PieChart>
                      </ResponsiveContainer>
                      <div className="absolute inset-0 flex flex-col items-center justify-center">
                        <span className="text-4xl xl:text-6xl font-black tracking-tighter">
                          {stats.rate}%
                        </span>
                        <span className="text-[10px] font-black text-slate-500 uppercase tracking-widest mt-2">
                          Operational
                        </span>
                      </div>
                    </div>
                  </div>
                </div>

                <div
                  className={`${theme.card} rounded-[2.5rem] p-8 xl:p-14 border relative overflow-hidden group`}
                >
                  <div className="absolute top-0 right-0 w-64 h-64 bg-emerald-500/5 blur-[80px] rounded-full -translate-y-1/2 translate-x-1/2"></div>
                  <div className="relative z-10">
                    <div className="flex flex-col xl:flex-row xl:items-center justify-between gap-6 mb-8 xl:mb-12">
                      <div className="max-w-2xl">
                        <h2 className="text-2xl xl:text-4xl font-black tracking-tighter flex items-center gap-3">
                          <Sparkles className="text-emerald-500" size={24} />{" "}
                          Strategic Intel Report
                        </h2>
                        <p className="text-xs text-slate-500 font-bold uppercase tracking-widest mt-2">
                          Deep Learning Analysis
                        </p>
                      </div>
                      <button
                        onClick={generateReport}
                        disabled={analyzing}
                        className="px-8 xl:px-12 py-4 xl:py-6 bg-emerald-500 hover:bg-emerald-400 text-white rounded-xl text-sm xl:text-base font-black uppercase tracking-widest transition-all shadow-xl shadow-emerald-500/30 disabled:opacity-50 flex items-center justify-center gap-4 shrink-0"
                      >
                        {analyzing ? (
                          <RefreshCw className="animate-spin" size={20} />
                        ) : (
                          <BrainCircuit size={20} />
                        )}
                        {analyzing ? "Processing..." : "Generate AI Insight"}
                      </button>
                    </div>

                    {aiAnalysis ? (
                      <div
                        className={`p-6 xl:p-12 rounded-[2rem] border text-base xl:text-xl animate-in slide-in-from-bottom-4 duration-500 ${
                          isDarkMode
                            ? "bg-slate-900/40 border-white/5"
                            : "bg-slate-50 border-slate-200"
                        }`}
                      >
                        <div className="flex gap-4 xl:gap-8">
                          <Info
                            size={24}
                            className="text-emerald-500 shrink-0"
                          />
                          <p className="whitespace-pre-wrap leading-relaxed xl:leading-[1.8]">
                            {aiAnalysis}
                          </p>
                        </div>
                      </div>
                    ) : (
                      <div className="py-16 xl:py-24 text-center border-2 border-dashed border-slate-800/20 rounded-[2rem] flex flex-col items-center">
                        <MessageSquare
                          className="text-slate-800/30 mb-4"
                          size={48}
                        />
                        <p className="text-[10px] font-black text-slate-500 uppercase tracking-widest">
                          System Ready for Analysis
                        </p>
                      </div>
                    )}
                  </div>
                </div>
              </>
            )}

            {activeTab === "table" && (
              <div className="space-y-6 animate-in fade-in duration-500">
                <div
                  className={`${theme.card} rounded-[2rem] border overflow-hidden`}
                >
                  <div className="overflow-x-auto overflow-y-hidden">
                    <table className="w-full text-left min-w-[1000px]">
                      <thead
                        className={`text-[10px] font-black uppercase tracking-widest ${
                          isDarkMode
                            ? "bg-white/5 text-slate-500"
                            : "bg-slate-50 text-slate-400"
                        }`}
                      >
                        <tr>
                          <th className="px-8 py-8">Node</th>
                          <th className="px-8 py-8">Task ID</th>
                          <th className="px-8 py-8">Action Matrix</th>
                          <th className="px-8 py-8 text-center">Status</th>
                          <th className="px-8 py-8"></th>
                        </tr>
                      </thead>
                      <tbody
                        className={`divide-y ${
                          isDarkMode ? "divide-white/5" : "divide-slate-100"
                        }`}
                      >
                        {filteredData.map((item) => (
                          <tr
                            key={item.id}
                            onClick={() => handleSelectItem(item)}
                            className="group hover:bg-emerald-500/5 transition-colors cursor-pointer"
                          >
                            <td className="px-8 py-6 xl:py-8">
                              <div className="font-black text-lg xl:text-xl group-hover:text-emerald-500 transition-colors truncate max-w-[200px]">
                                {item.ais1stTier}
                              </div>
                              <div className="flex items-center gap-2 mt-1 opacity-60">
                                <Hash size={14} className="text-emerald-500" />
                                <span className="text-xs font-mono font-black">
                                  {item.siteCode}
                                </span>
                              </div>
                            </td>
                            <td className="px-8 py-6 xl:py-8">
                              <div className="text-xs font-black text-emerald-500 mb-1">
                                {item.jobId}
                              </div>
                              <div className="flex items-center gap-2 text-xs font-bold opacity-70">
                                <Timer size={14} /> {item.date}
                              </div>
                            </td>
                            <td className="px-8 py-6 xl:py-8 max-w-sm">
                              <div className="text-sm xl:text-base font-bold leading-relaxed line-clamp-2">
                                {item.action}
                              </div>
                              <div className="text-[10px] font-black opacity-50 mt-1 flex items-center gap-2">
                                <MapPin size={12} /> {item.province}
                              </div>
                            </td>
                            <td className="px-8 py-6 xl:py-8 text-center">
                              <StatusPill status={item.status} />
                            </td>
                            <td className="px-8 py-6 xl:py-8 text-right">
                              <div className="inline-flex p-3 rounded-xl bg-slate-500/10 group-hover:bg-emerald-500 group-hover:text-white transition-all">
                                <ChevronRight size={20} />
                              </div>
                            </td>
                          </tr>
                        ))}
                      </tbody>
                    </table>
                  </div>
                </div>
              </div>
            )}
          </div>
        </div>
      </main>

      {selectedItem && (
        <div className="fixed inset-0 z-[300] flex justify-end">
          <div
            className="absolute inset-0 bg-black/80 backdrop-blur-md animate-in fade-in duration-300"
            onClick={() => setSelectedItem(null)}
          ></div>
          <div
            className={`w-full max-w-lg xl:max-w-2xl ${
              isDarkMode ? "bg-[#020617]" : "bg-white"
            } h-full relative p-8 xl:p-16 shadow-2xl animate-in slide-in-from-right duration-500 flex flex-col border-l ${
              isDarkMode ? "border-white/10" : "border-slate-200"
            }`}
          >
            <button
              onClick={() => setSelectedItem(null)}
              className="absolute top-8 right-8 p-3 rounded-xl hover:bg-red-500 hover:text-white transition-all bg-slate-500/10"
            >
              <X size={24} />
            </button>

            <div className="mb-10 mt-6 shrink-0">
              <span className="px-4 py-1.5 rounded-full bg-emerald-500/10 text-emerald-500 text-[10px] font-black uppercase tracking-widest border border-emerald-500/20">
                Operational Hub
              </span>
              <h2 className="text-3xl xl:text-5xl font-black tracking-tighter mt-6 leading-tight break-words">
                {selectedItem.ais1stTier}
              </h2>
              <p className="text-sm font-mono font-bold text-slate-500 mt-3">
                {selectedItem.jobId} // {selectedItem.siteCode}
              </p>
            </div>

            <div className="space-y-8 flex-1 overflow-y-auto custom-scrollbar pr-4">
              <DetailBlock label="Context" icon={MapPin}>
                <p className="font-black text-xl xl:text-2xl">
                  {selectedItem.province}, {selectedItem.region}
                </p>
                {selectedItem.lat && (
                  <p className="text-[10px] font-mono text-slate-500 mt-2">
                    GEO: {selectedItem.lat.toFixed(4)},{" "}
                    {selectedItem.lng.toFixed(4)}
                  </p>
                )}
              </DetailBlock>

              <DetailBlock label="Action Plan" icon={Zap}>
                <p className="text-base xl:text-lg font-bold leading-relaxed text-slate-400">
                  {selectedItem.action}
                </p>
              </DetailBlock>

              <DetailBlock label="Log" icon={FileText}>
                <div
                  className={`p-6 xl:p-8 rounded-2xl border ${
                    isDarkMode
                      ? "bg-white/5 border-white/5 text-slate-300"
                      : "bg-slate-50 border-slate-200 text-slate-600"
                  } text-sm xl:text-base leading-relaxed italic`}
                >
                  "{selectedItem.details}"
                </div>
              </DetailBlock>

              <div className="pt-8 border-t border-white/5">
                <h4 className="text-[10px] font-black text-emerald-500 uppercase tracking-widest mb-6 flex items-center gap-3">
                  <Sparkles size={18} /> AI Recs
                </h4>
                {advising ? (
                  <div className="py-10 flex flex-col items-center">
                    <RefreshCw
                      className="animate-spin text-emerald-500 mb-4"
                      size={24}
                    />
                    <span className="text-[10px] font-black uppercase text-slate-500 animate-pulse">
                      Analyzing...
                    </span>
                  </div>
                ) : (
                  siteAdvice && (
                    <div
                      className={`p-6 xl:p-10 rounded-3xl text-sm xl:text-lg leading-relaxed ${
                        isDarkMode
                          ? "bg-emerald-500/10 text-emerald-100 border border-emerald-500/30"
                          : "bg-emerald-50 text-emerald-900 border border-emerald-200 shadow-xl"
                      }`}
                    >
                      {siteAdvice}
                    </div>
                  )
                )}
              </div>
            </div>

            <div className="mt-10 pt-8 border-t border-white/5 shrink-0">
              <button
                onClick={() => setSelectedItem(null)}
                className="w-full py-4 xl:py-6 bg-emerald-500 hover:bg-emerald-400 text-white rounded-2xl font-black uppercase tracking-widest transition-all text-xs xl:text-sm"
              >
                Authorize Sync
              </button>
            </div>
          </div>
        </div>
      )}

      <style>{`
        .custom-scrollbar::-webkit-scrollbar { width: 6px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: rgba(16, 185, 129, 0.3); border-radius: 10px; }
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; letter-spacing: -0.01em; -webkit-font-smoothing: antialiased; }
        .leaflet-container { background: transparent !important; }
        .leaflet-popup-content-wrapper { border-radius: 16px; padding: 4px; box-shadow: 0 10px 30px rgba(0,0,0,0.15); font-family: 'Plus Jakarta Sans', sans-serif; }
      `}</style>
    </div>
  );
}

function SideLink({ active, icon: Icon, label, onClick, isDarkMode }) {
  return (
    <button
      onClick={onClick}
      className={`w-full flex items-center gap-4 px-6 py-4 rounded-2xl text-[14px] font-black transition-all group relative overflow-hidden ${
        active
          ? "bg-emerald-500 text-white shadow-lg shadow-emerald-500/30"
          : isDarkMode
          ? "text-slate-500 hover:text-slate-100 hover:bg-white/5"
          : "text-slate-500 hover:bg-slate-100"
      }`}
    >
      <Icon
        size={20}
        className={`shrink-0 transition-transform ${
          active ? "scale-110" : "group-hover:scale-110"
        }`}
      />
      <span className="truncate">{label}</span>
      {active && (
        <div className="absolute right-0 top-1/2 -translate-y-1/2 w-1.5 h-6 bg-white rounded-l-full"></div>
      )}
    </button>
  );
}

function SelectBox({ label, value, options, onChange, isDarkMode }) {
  return (
    <div className="space-y-2">
      <span className="text-[9px] font-black text-slate-500 uppercase tracking-widest ml-1">
        {label}
      </span>
      <div className="relative">
        <select
          value={value}
          onChange={(e) => onChange(e.target.value)}
          className={`w-full py-3.5 px-5 rounded-xl text-[12px] font-black outline-none border appearance-none transition-all cursor-pointer ${
            isDarkMode
              ? "bg-[#0f172a] border-white/10 text-slate-200 focus:border-emerald-500/50"
              : "bg-white border-slate-200 text-slate-800"
          }`}
        >
          {options.map((opt) => (
            <option key={opt} value={opt}>
              {opt}
            </option>
          ))}
        </select>
        <ChevronRight
          size={14}
          className="absolute right-4 top-1/2 -translate-y-1/2 rotate-90 pointer-events-none opacity-40"
        />
      </div>
    </div>
  );
}

function MiniStat({ label, value, icon: Icon, color, isDarkMode }) {
  const styles = {
    emerald: "text-emerald-500 bg-emerald-500/10",
    blue: "text-blue-500 bg-blue-500/10",
    amber: "text-amber-500 bg-amber-500/10",
    purple: "text-purple-500 bg-purple-500/10",
  };
  return (
    <div
      className={`p-6 xl:p-10 rounded-[2rem] border transition-all hover:-translate-y-1 ${
        isDarkMode
          ? "bg-[#0f172a] border-white/5"
          : "bg-white border-slate-200 shadow-sm"
      }`}
    >
      <div
        className={`w-10 h-10 xl:w-12 xl:h-12 rounded-xl ${styles[color]} flex items-center justify-center mb-6`}
      >
        <Icon size={20} className="xl:size-24" />
      </div>
      <p className="text-2xl xl:text-4xl font-black tracking-tighter truncate">
        {value}
      </p>
      <p className="text-[10px] font-black text-slate-500 uppercase tracking-widest mt-2 truncate">
        {label}
      </p>
    </div>
  );
}

function StatusPill({ status }) {
  const isDone = String(status).toLowerCase().includes("done");
  return (
    <div
      className={`inline-flex items-center gap-2 px-5 py-2 rounded-full text-[9px] font-black uppercase tracking-widest border transition-all ${
        isDone
          ? "bg-emerald-500/10 border-emerald-500/20 text-emerald-500"
          : "bg-amber-500/10 border-amber-500/20 text-amber-500"
      }`}
    >
      {isDone ? <CheckCircle2 size={12} /> : <Activity size={12} />}
      <span>{isDone ? "Synced" : "Active"}</span>
    </div>
  );
}

function DetailBlock({ label, icon: Icon, children }) {
  return (
    <div className="space-y-3">
      <div className="flex items-center gap-3 text-[10px] font-black text-slate-500 uppercase tracking-widest">
        <Icon size={18} /> <span>{label}</span>
      </div>
      <div className="pl-6 border-l border-emerald-500/20">{children}</div>
    </div>
  );
}
