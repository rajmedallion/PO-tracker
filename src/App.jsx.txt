import React, { useEffect, useMemo, useRef, useState } from "react";
import { createClient } from "@supabase/supabase-js";

const defaultStages = [
  "In-house Manufacturing",
  "Outsource Manufacturing",
  "Return from Outsource Vendor",
  "Sent to Paint",
  "Return from Paint",
  "Ready to Ship",
  "Delivered to Customer",
];

const defaultStageText = defaultStages.join("\n");

function createId() {
  if (typeof crypto !== "undefined" && typeof crypto.randomUUID === "function") {
    return crypto.randomUUID();
  }
  return `${Date.now()}-${Math.random().toString(36).slice(2)}`;
}

function today() {
  return new Date().toISOString().slice(0, 10);
}

function normalizeStagesText(value) {
  return String(value || "")
    .split("\n")
    .map((item) => item.trim())
    .filter(Boolean);
}

function badgeStyle(stage, stages) {
  const isDone = stage === stages[stages.length - 1];
  const isShipping = /ship|deliver/i.test(stage);
  const isPaint = /paint/i.test(stage);
  if (isDone) return "bg-emerald-100 text-emerald-700 border-emerald-200";
  if (isShipping) return "bg-blue-100 text-blue-700 border-blue-200";
  if (isPaint) return "bg-amber-100 text-amber-700 border-amber-200";
  return "bg-slate-100 text-slate-700 border-slate-200";
}

function StatCard({ title, value, subtitle }) {
  return (
    <div className="rounded-2xl border border-slate-200 bg-white p-5 shadow-sm">
      <div className="text-sm text-slate-500">{title}</div>
      <div className="mt-2 text-3xl font-semibold text-slate-900">{value}</div>
      <div className="mt-1 text-xs text-slate-500">{subtitle}</div>
    </div>
  );
}

function SectionCard({ title, children, actions }) {
  return (
    <div className="rounded-3xl border border-slate-200 bg-white shadow-sm">
      <div className="flex flex-col gap-3 border-b border-slate-100 px-6 py-5 md:flex-row md:items-center md:justify-between">
        <h2 className="text-lg font-semibold text-slate-900">{title}</h2>
        {actions ? <div className="flex flex-wrap gap-2">{actions}</div> : null}
      </div>
      <div className="p-6">{children}</div>
    </div>
  );
}

export default function POTrackerApp() {
  const [supabaseUrl, setSupabaseUrl] = useState("");
  const [supabaseKey, setSupabaseKey] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [client, setClient] = useState(null);
  const [orders, setOrders] = useState([]);
  const [po, setPo] = useState("");
  const [customer, setCustomer] = useState("");
  const [stages, setStages] = useState(defaultStages);
  const [stageDraft, setStageDraft] = useState(defaultStageText);
  const [stage, setStage] = useState(defaultStages[0]);
  const [search, setSearch] = useState("");
  const [status, setStatus] = useState("Not connected");
  const [loading, setLoading] = useState(false);
  const [isSignedIn, setIsSignedIn] = useState(false);
  const didLoadSharedRef = useRef(false);

  useEffect(() => {
    if (!supabaseUrl || !supabaseKey) {
      setClient(null);
      setIsSignedIn(false);
      setStatus("Not connected");
      return;
    }

    try {
      const nextClient = createClient(supabaseUrl.trim(), supabaseKey.trim(), {
        auth: {
          persistSession: true,
          autoRefreshToken: true,
        },
      });
      setClient(nextClient);
      setStatus("Supabase connected");
    } catch {
      setClient(null);
      setIsSignedIn(false);
      setStatus("Connection failed");
    }
  }, [supabaseUrl, supabaseKey]);

  useEffect(() => {
    if (!client) return;

    let active = true;

    async function bootstrap() {
      try {
        const { data } = await client.auth.getSession();
        if (!active) return;
        const signedIn = Boolean(data?.session);
        setIsSignedIn(signedIn);
        if (signedIn) {
          await Promise.all([loadStages(client), loadOrders(client)]);
        }
      } catch {
        if (active) {
          setStatus("Connection ready");
        }
      }
    }

    bootstrap();

    const { data: listener } = client.auth.onAuthStateChange(async (_event, session) => {
      if (!active) return;
      const signedIn = Boolean(session);
      setIsSignedIn(signedIn);
      if (!signedIn) {
        didLoadSharedRef.current = false;
        setOrders([]);
        setStages(defaultStages);
        setStageDraft(defaultStageText);
        setStage(defaultStages[0]);
        setStatus("Signed out");
        return;
      }
      if (!didLoadSharedRef.current) {
        didLoadSharedRef.current = true;
        await Promise.all([loadStages(client), loadOrders(client)]);
      }
    });

    return () => {
      active = false;
      listener?.subscription?.unsubscribe?.();
    };
  }, [client]);

  async function signUp() {
    if (!client) return;
    setLoading(true);
    try {
      const { error } = await client.auth.signUp({ email: email.trim(), password });
      setStatus(error ? error.message : "Login created");
    } finally {
      setLoading(false);
    }
  }

  async function signIn() {
    if (!client) return;
    setLoading(true);
    try {
      const { error } = await client.auth.signInWithPassword({
        email: email.trim(),
        password,
      });
      if (error) {
        setStatus(error.message);
        return;
      }
      didLoadSharedRef.current = true;
      setIsSignedIn(true);
      setStatus("Signed in");
      await Promise.all([loadStages(client), loadOrders(client)]);
    } finally {
      setLoading(false);
    }
  }

  async function loadStages(activeClient = client) {
    if (!activeClient) return;
    const { data, error } = await activeClient
      .from("po_tracker_stages")
      .select("name, position")
      .order("position", { ascending: true });

    if (error) {
      setStatus(error.message);
      return;
    }

    const nextStages = Array.isArray(data) && data.length ? data.map((item) => item.name).filter(Boolean) : defaultStages;
    setStages(nextStages);
    setStageDraft(nextStages.join("\n"));
    setStage((current) => (nextStages.includes(current) ? current : nextStages[0]));
  }

  async function loadOrders(activeClient = client) {
    if (!activeClient) return;
    setLoading(true);
    try {
      const { data, error } = await activeClient
        .from("po_tracker_orders")
        .select("*")
        .order("updated_at", { ascending: false });
      if (error) {
        setStatus(error.message);
      } else {
        setOrders(Array.isArray(data) ? data : []);
        setStatus("Orders loaded");
      }
    } finally {
      setLoading(false);
    }
  }

  async function saveStages() {
    const nextStages = normalizeStagesText(stageDraft);
    if (!client || nextStages.length === 0) {
      setStatus("Add at least one stage");
      return;
    }
    setLoading(true);
    try {
      const deleteResult = await client.from("po_tracker_stages").delete().gte("position", 0);
      if (deleteResult.error) {
        setStatus(deleteResult.error.message);
        return;
      }
      const insertResult = await client
        .from("po_tracker_stages")
        .insert(nextStages.map((name, position) => ({ name, position })));
      if (insertResult.error) {
        setStatus(insertResult.error.message);
        return;
      }
      setStages(nextStages);
      setStageDraft(nextStages.join("\n"));
      setStage((current) => (nextStages.includes(current) ? current : nextStages[0]));
      setStatus("Stages updated");
    } finally {
      setLoading(false);
    }
  }

  async function addOrder() {
    if (!client || !po.trim() || !customer.trim()) return;
    setLoading(true);
    try {
      const { error } = await client.from("po_tracker_orders").insert({
        id: createId(),
        po_number: po.trim(),
        customer_name: customer.trim(),
        stage,
        parts: [],
        updated_at: today(),
        updated_by: email || "manual",
        location: "",
        notes: "",
      });
      if (error) {
        setStatus(error.message);
      } else {
        setPo("");
        setCustomer("");
        setStage(stages[0] || defaultStages[0]);
        setStatus("Order added");
        await loadOrders(client);
      }
    } finally {
      setLoading(false);
    }
  }

  const filteredOrders = useMemo(() => {
    const q = search.trim().toLowerCase();
    if (!q) return orders;
    return orders.filter((o) =>
      [o.po_number, o.customer_name, o.stage, o.updated_by, o.location, o.notes]
        .join(" ")
        .toLowerCase()
        .includes(q)
    );
  }, [orders, search]);

  const activeOrders = orders.length;
  const inTransit = orders.filter((o) => /outsource|paint|ship|deliver/i.test(o.stage || "")).length;
  const readyToShip = orders.filter((o) => /ready to ship/i.test(o.stage || "")).length;
  const delivered = orders.filter((o) => o.stage === stages[stages.length - 1]).length;

  return (
    <div className="min-h-screen bg-slate-50 px-4 py-8 text-slate-900 md:px-8">
      <div className="mx-auto max-w-7xl space-y-6">
        <div className="flex flex-col gap-3 md:flex-row md:items-end md:justify-between">
          <div>
            <h1 className="text-3xl font-bold tracking-tight">PO Tracker</h1>
            <p className="mt-2 text-sm text-slate-500">
              Purchase order tracking for manufacturing, outsourcing, paint, shipping, and delivery.
            </p>
          </div>
          <div className="rounded-2xl border border-slate-200 bg-white px-4 py-3 text-sm shadow-sm">
            <span className="font-medium text-slate-700">Status:</span> {status}
          </div>
        </div>

        <div className="grid gap-4 md:grid-cols-2 xl:grid-cols-4">
          <StatCard title="Active Orders" value={activeOrders} subtitle="Total tracked purchase orders" />
          <StatCard title="Transit / Outside" value={inTransit} subtitle="Outsource, paint, and shipping stages" />
          <StatCard title="Ready to Ship" value={readyToShip} subtitle="Orders prepared for dispatch" />
          <StatCard title="Delivered" value={delivered} subtitle="Completed customer deliveries" />
        </div>

        <div className="grid gap-6 xl:grid-cols-[1.05fr_1.35fr]">
          <SectionCard
            title="Connection & Login"
            actions={
              <>
                <button
                  onClick={signUp}
                  disabled={loading}
                  className="rounded-xl border border-slate-200 bg-white px-4 py-2 text-sm font-medium text-slate-700 transition hover:bg-slate-50 disabled:opacity-50"
                >
                  Create Login
                </button>
                <button
                  onClick={signIn}
                  disabled={loading}
                  className="rounded-xl bg-slate-900 px-4 py-2 text-sm font-medium text-white transition hover:bg-slate-800 disabled:opacity-50"
                >
                  {isSignedIn ? "Signed In" : "Sign In"}
                </button>
              </>
            }
          >
            <div className="grid gap-4">
              <div className="grid gap-4 md:grid-cols-2">
                <div>
                  <label className="mb-2 block text-sm font-medium text-slate-700">Supabase Project URL</label>
                  <input
                    className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none ring-0 transition focus:border-slate-400"
                    placeholder="https://your-project.supabase.co"
                    value={supabaseUrl}
                    onChange={(e) => setSupabaseUrl(e.target.value)}
                  />
                </div>
                <div>
                  <label className="mb-2 block text-sm font-medium text-slate-700">Supabase Publishable / Anon Key</label>
                  <input
                    className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none ring-0 transition focus:border-slate-400"
                    placeholder="sb_publishable_..."
                    value={supabaseKey}
                    onChange={(e) => setSupabaseKey(e.target.value)}
                  />
                </div>
              </div>
              <div className="grid gap-4 md:grid-cols-2">
                <div>
                  <label className="mb-2 block text-sm font-medium text-slate-700">Email</label>
                  <input
                    className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none ring-0 transition focus:border-slate-400"
                    placeholder="staff@company.com"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                  />
                </div>
                <div>
                  <label className="mb-2 block text-sm font-medium text-slate-700">Password</label>
                  <input
                    type="password"
                    className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none ring-0 transition focus:border-slate-400"
                    placeholder="Password"
                    value={password}
                    onChange={(e) => setPassword(e.target.value)}
                  />
                </div>
              </div>
            </div>
          </SectionCard>

          <SectionCard
            title="Workflow Stages"
            actions={
              <button
                onClick={saveStages}
                disabled={loading}
                className="rounded-xl border border-slate-200 bg-white px-4 py-2 text-sm font-medium text-slate-700 transition hover:bg-slate-50 disabled:opacity-50"
              >
                Save Stages
              </button>
            }
          >
            <div className="grid gap-4">
              <div>
                <label className="mb-2 block text-sm font-medium text-slate-700">Editable Stage Names</label>
                <textarea
                  className="min-h-[220px] w-full rounded-2xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none transition focus:border-slate-400"
                  value={stageDraft}
                  onChange={(e) => setStageDraft(e.target.value)}
                  placeholder="One stage per line"
                />
              </div>
              <div className="text-xs text-slate-500">Add one stage per line. Save to update the stage list used in the app.</div>
            </div>
          </SectionCard>

          <SectionCard
            title="Create Purchase Order"
            actions={
              <button
                onClick={addOrder}
                disabled={loading}
                className="rounded-xl bg-slate-900 px-4 py-2 text-sm font-medium text-white transition hover:bg-slate-800 disabled:opacity-50"
              >
                Add Order
              </button>
            }
          >
            <div className="grid gap-4 md:grid-cols-3">
              <div>
                <label className="mb-2 block text-sm font-medium text-slate-700">PO Number</label>
                <input
                  className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none transition focus:border-slate-400"
                  placeholder="PO-1001"
                  value={po}
                  onChange={(e) => setPo(e.target.value)}
                />
              </div>
              <div>
                <label className="mb-2 block text-sm font-medium text-slate-700">Customer</label>
                <input
                  className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none transition focus:border-slate-400"
                  placeholder="Customer name"
                  value={customer}
                  onChange={(e) => setCustomer(e.target.value)}
                />
              </div>
              <div>
                <label className="mb-2 block text-sm font-medium text-slate-700">Current Stage</label>
                <select
                  className="w-full rounded-xl border border-slate-200 bg-white px-4 py-3 text-sm outline-none transition focus:border-slate-400"
                  value={stage}
                  onChange={(e) => setStage(e.target.value)}
                >
                  {stages.map((s) => (
                    <option key={s} value={s}>
                      {s}
                    </option>
                  ))}
                </select>
              </div>
            </div>
          </SectionCard>
        </div>

        <SectionCard
          title="Order List"
          actions={
            <>
              <input
                className="w-full rounded-xl border border-slate-200 bg-white px-4 py-2.5 text-sm outline-none transition focus:border-slate-400 md:w-64"
                placeholder="Search by PO, customer, stage"
                value={search}
                onChange={(e) => setSearch(e.target.value)}
              />
              <button
                onClick={() => loadOrders(client)}
                disabled={loading}
                className="rounded-xl border border-slate-200 bg-white px-4 py-2 text-sm font-medium text-slate-700 transition hover:bg-slate-50 disabled:opacity-50"
              >
                Refresh
              </button>
            </>
          }
        >
          <div className="overflow-hidden rounded-2xl border border-slate-200">
            <div className="overflow-x-auto">
              <table className="min-w-full border-collapse bg-white">
                <thead>
                  <tr className="bg-slate-100 text-left text-sm text-slate-600">
                    <th className="px-4 py-3 font-semibold">PO Number</th>
                    <th className="px-4 py-3 font-semibold">Customer</th>
                    <th className="px-4 py-3 font-semibold">Stage</th>
                    <th className="px-4 py-3 font-semibold">Updated Date</th>
                    <th className="px-4 py-3 font-semibold">Updated By</th>
                  </tr>
                </thead>
                <tbody>
                  {filteredOrders.length === 0 ? (
                    <tr>
                      <td colSpan={5} className="px-4 py-10 text-center text-sm text-slate-500">
                        No orders found.
                      </td>
                    </tr>
                  ) : (
                    filteredOrders.map((o) => (
                      <tr key={o.id} className="border-t border-slate-100 text-sm">
                        <td className="px-4 py-3 font-medium text-slate-900">{o.po_number}</td>
                        <td className="px-4 py-3 text-slate-700">{o.customer_name}</td>
                        <td className="px-4 py-3">
                          <span className={`inline-flex rounded-full border px-3 py-1 text-xs font-medium ${badgeStyle(o.stage || "", stages)}`}>
                            {o.stage}
                          </span>
                        </td>
                        <td className="px-4 py-3 text-slate-600">{o.updated_at || "—"}</td>
                        <td className="px-4 py-3 text-slate-600">{o.updated_by || "—"}</td>
                      </tr>
                    ))
                  )}
                </tbody>
              </table>
            </div>
          </div>
        </SectionCard>
      </div>
    </div>
  );
}
