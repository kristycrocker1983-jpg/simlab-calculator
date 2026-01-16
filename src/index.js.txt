import React, { useState, useEffect } from 'react';
import { BarChart, Bar, LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, PieChart, Pie, Cell } from 'recharts';
import { Plus, Trash2, Eye, EyeOff } from 'lucide-react';

export default function SimLabCalculator() {
  const [events, setEvents] = useState([]);
  const [formData, setFormData] = useState({
    date: '',
    program: '',
    semester: '',
    scenario: '',
    staff: '',
    phase: '',
    type: '',
    room: '',
    numLearners: '',
    technology: '',
    status: 'Planned',
    duration: ''
  });
  const [activeTab, setActiveTab] = useState('input');
  const [loading, setLoading] = useState(false);

  // Configuration hours
  const configHours = {
    'PLANNING-NEW': 4.5,
    'PLANNING-EXISTING': 3,
    'SETUP-FIRST': 2,
    'SETUP-PROCEEDING': 0,
    'RESET': 0.5,
    'TEARDOWN': 2
  };

  // Load data from persistent storage on mount
  useEffect(() => {
    loadData();
  }, []);

  const loadData = async () => {
    try {
      setLoading(true);
      const result = await window.storage.list('simlab:');
      if (result && result.keys) {
        const loadedEvents = [];
        for (const key of result.keys) {
          const data = await window.storage.get(key);
          if (data) {
            loadedEvents.push(JSON.parse(data.value));
          }
        }
        setEvents(loadedEvents.sort((a, b) => new Date(b.date) - new Date(a.date)));
      }
    } catch (err) {
      console.log('Storage initialized - ready to save events');
    }
    setLoading(false);
  };

  const calculateHours = (phase, type, duration) => {
    if (phase === 'RUN') return parseFloat(duration) || 0;
    if (phase === 'RESET') return 0.5;
    if (phase === 'TEARDOWN') return 2;
    
    const key = `${phase}-${type}`;
    return configHours[key] || 0;
  };

  const handleAddEvent = async () => {
    if (!formData.scenario || !formData.phase || !formData.date) {
      alert('Please fill in Scenario, Phase, and Date');
      return;
    }

    if ((formData.phase === 'PLANNING' || formData.phase === 'SETUP') && !formData.type) {
      alert(`Please select Type for ${formData.phase} phase`);
      return;
    }

    if (formData.phase === 'RUN' && !formData.duration) {
      alert('Please enter Duration for RUN phase');
      return;
    }

    const hours = calculateHours(formData.phase, formData.type, formData.duration);
    
    const newEvent = {
      id: Date.now().toString(),
      date: formData.date,
      program: formData.program,
      semester: formData.semester,
      scenario: formData.scenario,
      staff: formData.staff,
      phase: formData.phase,
      type: formData.type,
      room: formData.room,
      numLearners: formData.numLearners,
      technology: formData.technology,
      status: formData.status,
      duration: formData.phase === 'RUN' ? parseFloat(formData.duration) : 0,
      hours: hours,
      createdAt: new Date().toISOString()
    };

    try {
      await window.storage.set(`simlab:${newEvent.id}`, JSON.stringify(newEvent));
      setEvents([newEvent, ...events]);
      setFormData({
        date: '',
        program: '',
        semester: '',
        scenario: '',
        staff: '',
        phase: '',
        type: '',
        room: '',
        numLearners: '',
        technology: '',
        status: 'Planned',
        duration: ''
      });
      alert('Event added successfully!');
    } catch (err) {
      alert('Error saving event: ' + err.message);
    }
  };

  const handleDeleteEvent = async (id) => {
    if (window.confirm('Are you sure you want to delete this event?')) {
      try {
        await window.storage.delete(`simlab:${id}`);
        setEvents(events.filter(e => e.id !== id));
      } catch (err) {
        alert('Error deleting event: ' + err.message);
      }
    }
  };

  // Calculate analytics
  const totalHours = events.reduce((sum, e) => sum + e.hours, 0);
  const scenarioStats = {};
  const phaseStats = {};
  const monthlyStats = {};

  events.forEach(event => {
    scenarioStats[event.scenario] = (scenarioStats[event.scenario] || 0) + event.hours;
    phaseStats[event.phase] = (phaseStats[event.phase] || 0) + event.hours;
    
    const month = event.date.substring(0, 7);
    monthlyStats[month] = (monthlyStats[month] || 0) + event.hours;
  });

  const monthlyData = Object.entries(monthlyStats)
    .sort()
    .map(([month, hours]) => ({ month, hours: parseFloat(hours.toFixed(2)) }));

  const phaseData = Object.entries(phaseStats).map(([phase, hours]) => ({
    name: phase,
    value: parseFloat(hours.toFixed(2)),
    color: { 'PLANNING': '#3b82f6', 'SETUP': '#10b981', 'RUN': '#f59e0b', 'RESET': '#8b5cf6', 'TEARDOWN': '#ef4444' }[phase]
  }));

  // TABLE VIEW
  const TableView = () => (
    <div className="overflow-x-auto">
      <table className="w-full text-sm border-collapse">
        <thead>
          <tr className="bg-blue-200">
            <th className="border border-blue-300 px-3 py-2 text-left">Date</th>
            <th className="border border-blue-300 px-3 py-2 text-left">Scenario</th>
            <th className="border border-blue-300 px-3 py-2 text-left">Staff</th>
            <th className="border border-blue-300 px-3 py-2 text-center">Phase</th>
            <th className="border border-blue-300 px-3 py-2 text-center">Type</th>
            <th className="border border-blue-300 px-3 py-2 text-center">Room</th>
            <th className="border border-blue-300 px-3 py-2 text-center">Status</th>
            <th className="border border-blue-300 px-3 py-2 text-right font-bold">Hours</th>
            <th className="border border-blue-300 px-3 py-2 text-center">Action</th>
          </tr>
        </thead>
        <tbody>
          {events.map((event, idx) => (
            <tr key={event.id} className={idx % 2 === 0 ? 'bg-white' : 'bg-blue-50'}>
              <td className="border border-blue-200 px-3 py-2 text-sm">{event.date}</td>
              <td className="border border-blue-200 px-3 py-2 font-semibold">{event.scenario}</td>
              <td className="border border-blue-200 px-3 py-2">{event.staff}</td>
              <td className="border border-blue-200 px-3 py-2 text-center text-xs font-semibold">{event.phase}</td>
              <td className="border border-blue-200 px-3 py-2 text-center text-xs">{event.type || '—'}</td>
              <td className="border border-blue-200 px-3 py-2 text-center text-sm">{event.room}</td>
              <td className="border border-blue-200 px-3 py-2 text-center text-xs">
                <span className={`px-2 py-1 rounded ${
                  event.status === 'Complete' ? 'bg-green-200 text-green-800' :
                  event.status === 'In Progress' ? 'bg-yellow-200 text-yellow-800' :
                  'bg-gray-200 text-gray-800'
                }`}>
                  {event.status}
                </span>
              </td>
              <td className="border border-blue-200 px-3 py-2 text-right font-semibold">{event.hours.toFixed(1)}</td>
              <td className="border border-blue-200 px-3 py-2 text-center">
                <button onClick={() => handleDeleteEvent(event.id)} className="text-red-600 hover:text-red-800">
                  <Trash2 size={16} />
                </button>
              </td>
            </tr>
          ))}
        </tbody>
        <tfoot>
          <tr className="bg-blue-300 font-bold">
            <td colSpan="7" className="border border-blue-300 px-3 py-2 text-right">TOTAL:</td>
            <td className="border border-blue-300 px-3 py-2 text-right">{totalHours.toFixed(1)} hours</td>
            <td></td>
          </tr>
        </tfoot>
      </table>
    </div>
  );

  // DASHBOARD VIEW
  const DashboardView = () => (
    <div className="space-y-4">
      <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
        <div className="bg-white p-4 rounded border border-green-200">
          <p className="text-sm text-gray-600">Total Hours</p>
          <p className="text-3xl font-bold text-green-600">{totalHours.toFixed(1)}</p>
          <p className="text-xs text-gray-500 mt-1">{events.length} events tracked</p>
        </div>
        <div className="bg-white p-4 rounded border border-green-200">
          <p className="text-sm text-gray-600">Unique Scenarios</p>
          <p className="text-3xl font-bold text-green-600">{Object.keys(scenarioStats).length}</p>
        </div>
        <div className="bg-white p-4 rounded border border-green-200">
          <p className="text-sm text-gray-600">Average per Event</p>
          <p className="text-3xl font-bold text-green-600">{(totalHours / events.length || 0).toFixed(1)}</p>
        </div>
      </div>

      {monthlyData.length > 0 && (
        <div className="bg-white p-4 rounded border border-green-200">
          <h4 className="font-semibold text-sm mb-3">Hours by Month</h4>
          <ResponsiveContainer width="100%" height={300}>
            <BarChart data={monthlyData}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="month" />
              <YAxis />
              <Tooltip />
              <Bar dataKey="hours" fill="#10b981" />
            </BarChart>
          </ResponsiveContainer>
        </div>
      )}

      {phaseData.length > 0 && (
        <div className="bg-white p-4 rounded border border-green-200">
          <h4 className="font-semibold text-sm mb-3">Hours by Phase</h4>
          <ResponsiveContainer width="100%" height={300}>
            <PieChart>
              <Pie data={phaseData} cx="50%" cy="50%" labelLine={false} label={({ name, value }) => `${name}: ${value}h`} outerRadius={100} fill="#8884d8" dataKey="value">
                {phaseData.map((entry, index) => (
                  <Cell key={`cell-${index}`} fill={entry.color} />
                ))}
              </Pie>
            </PieChart>
          </ResponsiveContainer>
        </div>
      )}

      {Object.keys(scenarioStats).length > 0 && (
        <div className="bg-white p-4 rounded border border-green-200">
          <h4 className="font-semibold text-sm mb-3">Hours by Scenario</h4>
          <div className="space-y-2">
            {Object.entries(scenarioStats)
              .sort((a, b) => b[1] - a[1])
              .map(([scenario, hours]) => (
                <div key={scenario} className="flex justify-between items-center">
                  <span className="text-sm">{scenario}</span>
                  <span className="font-semibold text-green-600">{hours.toFixed(1)} hrs</span>
                </div>
              ))}
          </div>
        </div>
      )}
    </div>
  );

  // REPORT VIEW
  const ReportView = () => (
    <div className="bg-white p-8 rounded border border-purple-200 space-y-6">
      <div>
        <h2 className="font-bold text-2xl mb-2">Simulation Lab Hours Report</h2>
        <p className="text-sm text-gray-600">Spring Semester 2026</p>
        <p className="text-sm text-gray-600">Generated: {new Date().toLocaleDateString()}</p>
      </div>

      <div className="grid grid-cols-3 gap-4">
        <div className="bg-purple-50 p-3 rounded">
          <p className="text-xs text-gray-600">Total Hours</p>
          <p className="text-2xl font-bold text-purple-600">{totalHours.toFixed(1)}</p>
        </div>
        <div className="bg-purple-50 p-3 rounded">
          <p className="text-xs text-gray-600">Total Events</p>
          <p className="text-2xl font-bold text-purple-600">{events.length}</p>
        </div>
        <div className="bg-purple-50 p-3 rounded">
          <p className="text-xs text-gray-600">Avg Event Length</p>
          <p className="text-2xl font-bold text-purple-600">{(totalHours / events.length || 0).toFixed(1)}h</p>
        </div>
      </div>

      {Object.keys(scenarioStats).length > 0 && (
        <div>
          <h3 className="font-semibold mb-3">Scenario Summary</h3>
          <div className="space-y-3">
            {Object.entries(scenarioStats)
              .sort((a, b) => b[1] - a[1])
              .map(([scenario, hours]) => {
                const scenarioEvents = events.filter(e => e.scenario === scenario);
                return (
                  <div key={scenario} className="border-l-4 border-purple-400 pl-3">
                    <p className="font-semibold text-sm">{scenario}</p>
                    <p className="text-sm text-gray-600">{scenarioEvents.length} events • {hours.toFixed(1)} hours total</p>
                  </div>
                );
              })}
          </div>
        </div>
      )}

      {phaseData.length > 0 && (
        <div>
          <h3 className="font-semibold mb-3">Hours by Phase</h3>
          <div className="space-y-2">
            {phaseData.map(phase => (
              <div key={phase.name} className="flex justify-between">
                <span className="text-sm">{phase.name}</span>
                <span className="font-semibold">{phase.value.toFixed(1)} hrs</span>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );

  return (
    <div className="w-full max-w-6xl mx-auto p-4 bg-gray-50">
      <h1 className="text-2xl font-bold mb-1">Simulation Lab Hours Calculator</h1>
      <p className="text-gray-600 mb-6">Shared Tracker for You & Ashley</p>

      {/* Tabs */}
      <div className="flex gap-2 mb-6 border-b">
        <button
          onClick={() => setActiveTab('input')}
          className={`px-4 py-2 font-semibold ${activeTab === 'input' ? 'border-b-2 border-blue-600 text-blue-600' : 'text-gray-600'}`}
        >
          Input Data
        </button>
        <button
          onClick={() => setActiveTab('table')}
          className={`px-4 py-2 font-semibold ${activeTab === 'table' ? 'border-b-2 border-blue-600 text-blue-600' : 'text-gray-600'}`}
        >
          Table View
        </button>
        <button
          onClick={() => setActiveTab('dashboard')}
          className={`px-4 py-2 font-semibold ${activeTab === 'dashboard' ? 'border-b-2 border-blue-600 text-blue-600' : 'text-gray-600'}`}
        >
          Dashboard
        </button>
        <button
          onClick={() => setActiveTab('report')}
          className={`px-4 py-2 font-semibold ${activeTab === 'report' ? 'border-b-2 border-blue-600 text-blue-600' : 'text-gray-600'}`}
        >
          Report
        </button>
      </div>

      {/* INPUT TAB */}
      {activeTab === 'input' && (
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          <h2 className="font-bold text-lg mb-4">Add New Event</h2>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
            <input
              type="date"
              value={formData.date}
              onChange={(e) => setFormData({ ...formData, date: e.target.value })}
              className="border rounded px-3 py-2"
            />
            <input
              type="text"
              value={formData.scenario}
              onChange={(e) => setFormData({ ...formData, scenario: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Scenario (e.g., Cookie)"
            />
            <input
              type="text"
              value={formData.program}
              onChange={(e) => setFormData({ ...formData, program: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Program (e.g., Nursing)"
            />
            <input
              type="text"
              value={formData.semester}
              onChange={(e) => setFormData({ ...formData, semester: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Semester (e.g., 2nd Semester)"
            />
            <input
              type="text"
              value={formData.staff}
              onChange={(e) => setFormData({ ...formData, staff: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Staff Member"
            />
            <select
              value={formData.phase}
              onChange={(e) => setFormData({ ...formData, phase: e.target.value, type: '' })}
              className="border rounded px-3 py-2"
            >
              <option value="">Select Phase</option>
              <option value="PLANNING">PLANNING</option>
              <option value="SETUP">SETUP</option>
              <option value="RUN">RUN</option>
              <option value="RESET">RESET</option>
              <option value="TEARDOWN">TEARDOWN</option>
            </select>

            {(formData.phase === 'PLANNING' || formData.phase === 'SETUP') && (
              <select
                value={formData.type}
                onChange={(e) => setFormData({ ...formData, type: e.target.value })}
                className="border rounded px-3 py-2"
              >
                <option value="">Select Type</option>
                {formData.phase === 'PLANNING' && (
                  <>
                    <option value="NEW">NEW Scenario</option>
                    <option value="EXISTING">EXISTING Scenario</option>
                  </>
                )}
                {formData.phase === 'SETUP' && (
                  <>
                    <option value="FIRST">FIRST Run</option>
                    <option value="PROCEEDING">PROCEEDING Run</option>
                  </>
                )}
              </select>
            )}

            {formData.phase === 'RUN' && (
              <input
                type="number"
                step="0.5"
                value={formData.duration}
                onChange={(e) => setFormData({ ...formData, duration: e.target.value })}
                className="border rounded px-3 py-2"
                placeholder="Duration (hours)"
              />
            )}

            <input
              type="text"
              value={formData.room}
              onChange={(e) => setFormData({ ...formData, room: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Room"
            />
            <input
              type="text"
              value={formData.numLearners}
              onChange={(e) => setFormData({ ...formData, numLearners: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="# of Learners"
            />
            <input
              type="text"
              value={formData.technology}
              onChange={(e) => setFormData({ ...formData, technology: e.target.value })}
              className="border rounded px-3 py-2"
              placeholder="Technology"
            />
            <select
              value={formData.status}
              onChange={(e) => setFormData({ ...formData, status: e.target.value })}
              className="border rounded px-3 py-2"
            >
              <option value="Planned">Planned</option>
              <option value="In Progress">In Progress</option>
              <option value="Complete">Complete</option>
            </select>
          </div>

          <button
            onClick={handleAddEvent}
            className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 flex items-center gap-2"
          >
            <Plus size={18} /> Add Event
          </button>
        </div>
      )}

      {/* TABLE TAB */}
      {activeTab === 'table' && (
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          {loading ? (
            <p className="text-gray-600">Loading...</p>
          ) : events.length === 0 ? (
            <p className="text-gray-600">No events yet. Add some on the Input Data tab.</p>
          ) : (
            <TableView />
          )}
        </div>
      )}

      {/* DASHBOARD TAB */}
      {activeTab === 'dashboard' && (
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          {events.length === 0 ? (
            <p className="text-gray-600">No events yet. Add some on the Input Data tab.</p>
          ) : (
            <DashboardView />
          )}
        </div>
      )}

      {/* REPORT TAB */}
      {activeTab === 'report' && (
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          {events.length === 0 ? (
            <p className="text-gray-600">No events yet. Add some on the Input Data tab.</p>
          ) : (
            <ReportView />
          )}
        </div>
      )}
    </div>
  );
}