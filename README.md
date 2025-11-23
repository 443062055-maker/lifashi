import React, { useState, useRef, useEffect } from 'react';
import { 
  Scissors, User, ChevronRight, Camera, Save, RefreshCw, Ruler, Check, 
  ImageIcon, Undo2, AlertTriangle, Clock, Heart, ThumbsDown, Lock, Crown, 
  UploadCloud, Plus, X, Loader2
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, collection, addDoc, getDocs, query, orderBy, serverTimestamp } from 'firebase/firestore';
import { getStorage, ref, uploadBytes, getDownloadURL } from 'firebase/storage';

// --- Firebase 初始化 ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const storage = getStorage(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'haircut-talk';

// --- 数据配置 ---
const getImg = (text, bg = 'e2e8f0', fg = '475569') => 
  `https://placehold.co/400x300/${bg}/${fg}/png?text=${encodeURIComponent(text)}`;

const OPTIONS = {
  sides: [
    { id: 'natural', label: '自然保留', desc: '盖住耳朵或半耳，不推光', icon: '👂', image: getImg('自然保留\n(不露头皮)') },
    { id: 'trim', label: '修剪服帖', desc: '露出耳朵，鬓角修短但不推光', icon: '✂️', image: getImg('修剪服帖\n(露耳)') },
    { id: 'fade-low', label: '低位渐变', desc: '鬓角推白，向上自然过渡', icon: '📉', image: getImg('低位渐变\n(Low Fade)') },
    { id: 'fade-high', label: '美式铲青', desc: '两边推得很高很光', icon: '🇺🇸', image: getImg('高位铲青\n(High Fade)') },
    { id: 'two-block', label: '两分区/掏空', desc: '里面推短，外面头发盖下来', icon: '🧱', image: getImg('两分区\n(掏空)') },
  ],
  top: [
    { id: 'short', label: '寸头/短毛刺', desc: '无需打理，硬朗', icon: '🧊', image: getImg('寸头/毛刺') },
    { id: 'texture', label: '碎发/纹理', desc: '适合抓发蜡，有层次感', icon: '🌊', image: getImg('碎发纹理\n(需打理)') },
    { id: 'side-part', label: '分头/油头', desc: '二八/三七分', icon: '🤵', image: getImg('侧分/油头') },
    { id: 'perm', label: '烫发/卷发', desc: '钢夹烫/锡纸烫等', icon: '🌀', image: getImg('纹理烫/锡纸烫') },
    { id: 'long', label: '武士头/扎发', desc: '长发扎起来', icon: '🦁', image: getImg('长发/扎发') },
  ],
  bangs: [
    { id: 'none', label: '无/露额头', desc: '背头或飞机头', icon: '☀️', image: getImg('露额头\n(背头)') },
    { id: 'crop', label: '栗子头/齐刘海', desc: '短且齐，减龄', icon: '🌰', image: getImg('栗子头\n(齐刘海)') },
    { id: 'comma', label: '逗号/括号', desc: '韩式造型，发尾内扣', icon: '🇰🇷', image: getImg('韩式逗号') },
    { id: 'eyebrow', label: '碎盖/过眉', desc: '长刘海，遮盖额头', icon: '😎', image: getImg('碎盖\n(过眉)') },
  ],
  back: [
    { id: 'natural', label: '自然碎发', desc: '边界柔和', icon: '🌿', image: getImg('自然边界') },
    { id: 'square', label: '方形边界', desc: '硬朗直线', icon: '⬜', image: getImg('方形边界') },
    { id: 'taper', label: '渐变推短', desc: '由短到长过渡', icon: '📐', image: getImg('渐变收尾') },
  ],
  donts: [
    '切记不要打太薄',
    '刘海不要剪太短',
    '鬓角不要推太青',
    '不要用推子，全手剪',
    '头顶不要剪太短',
    '不要用太多发胶',
  ]
};

const CURRENT_LENGTHS = [
  { id: 'very_short', label: '贴皮/寸头', cm: '< 3cm', level: 1, avgCm: 1.5 },
  { id: 'short', label: '耳上短发', cm: '3-8cm', level: 2, avgCm: 5 },
  { id: 'medium', label: '盖耳/中发', cm: '8-15cm', level: 3, avgCm: 12 },
  { id: 'long', label: '及肩/长发', cm: '> 15cm', level: 4, avgCm: 20 },
];

const DEFAULT_STYLES = [
  { 
    id: 'buzz', name: '硬汉寸头', desc: '干净利落，无需打理', minLengthCm: 1,
    image: getImg('硬汉寸头\n(Style Ref)', '1e293b', 'ffffff'),
    config: { sides: OPTIONS.sides[3], top: OPTIONS.top[0], bangs: OPTIONS.bangs[0], back: OPTIONS.back[2] } 
  },
  { 
    id: 'crop', name: '栗子头', desc: '适合亚洲男生，修饰发际线', minLengthCm: 4,
    image: getImg('栗子头\n(Style Ref)', '1e293b', 'ffffff'),
    config: { sides: OPTIONS.sides[4], top: OPTIONS.top[0], bangs: OPTIONS.bangs[1], back: OPTIONS.back[0] } 
  },
  { 
    id: 'side_part', name: '商务侧分', desc: '成熟稳重，适合上班族', minLengthCm: 8,
    image: getImg('商务侧分\n(Style Ref)', '1e293b', 'ffffff'),
    config: { sides: OPTIONS.sides[2], top: OPTIONS.top[2], bangs: OPTIONS.bangs[0], back: OPTIONS.back[2] } 
  },
];

const GROWTH_RATE = 1.2;
const calculateGrowthTime = (currentLevel, targetMinCm) => {
  const current = CURRENT_LENGTHS.find(l => l.level === currentLevel);
  if (!current) return 0;
  const diff = targetMinCm - current.avgCm;
  return diff <= 0 ? 0 : Math.ceil(diff / GROWTH_RATE);
};

// --- 组件 ---

const ImageOptionCard = ({ item, selected, onClick, small = false }) => (
  <button
    onClick={onClick}
    className={`relative group overflow-hidden rounded-xl border-2 transition-all duration-200 text-left flex flex-col ${
      selected 
      ? 'border-blue-600 ring-1 ring-blue-600 shadow-md transform scale-[1.02]' 
      : 'border-slate-200 bg-white hover:border-blue-300'
    }`}
  >
    <div className={`w-full bg-slate-100 relative ${small ? 'h-24' : 'h-32'}`}>
       <img 
         src={item.image} 
         alt={item.label}
         className="w-full h-full object-cover opacity-90 group-hover:opacity-100 transition-opacity"
         onError={(e) => { e.target.onerror = null; e.target.src = `https://placehold.co/300x200?text=${item.label}`; }}
       />
       {selected && (
         <div className="absolute inset-0 bg-blue-900/20 flex items-center justify-center">
            <div className="bg-blue-600 text-white rounded-full p-1 shadow-lg"><Check size={20} strokeWidth={3} /></div>
         </div>
       )}
    </div>
    <div className="p-3 flex-1 flex flex-col justify-center bg-white">
      <div className="font-bold text-slate-800 text-sm flex items-center gap-1.5 mb-1">{item.label}</div>
      <div className="text-xs text-slate-500 leading-tight">{item.desc}</div>
    </div>
  </button>
);

const StepWizard = ({ children, currentStep }) => {
  return <div className="flex-1 overflow-y-auto bg-slate-50 pb-20">{children[currentStep]}</div>;
};

// --- Admin Upload Component ---
const AdminUploadScreen = ({ onBack, user }) => {
  const [uploading, setUploading] = useState(false);
  const [newStyle, setNewStyle] = useState({
    name: '', desc: '', minLengthCm: 5,
    config: { sides: OPTIONS.sides[0], top: OPTIONS.top[1], bangs: OPTIONS.bangs[0], back: OPTIONS.back[0] }
  });
  const [imageFile, setImageFile] = useState(null);
  const [imagePreview, setImagePreview] = useState(null);

  const handleFileChange = (e) => {
    const file = e.target.files[0];
    if (file) {
      setImageFile(file);
      const reader = new FileReader();
      reader.onloadend = () => setImagePreview(reader.result);
      reader.readAsDataURL(file);
    }
  };

  const handleConfigSelect = (category, value) => {
    setNewStyle(prev => ({ ...prev, config: { ...prev.config, [category]: value } }));
  };

  const handleSubmit = async () => {
    if (!newStyle.name || !imageFile || !user) return;
    setUploading(true);

    try {
      // 1. Upload Image
      const storageRef = ref(storage, `/artifacts/${appId}/public/data/images/${Date.now()}_${imageFile.name}`);
      const snapshot = await uploadBytes(storageRef, imageFile);
      const imageUrl = await getDownloadURL(snapshot.ref);

      // 2. Save Data
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'styles'), {
        ...newStyle,
        image: imageUrl,
        createdAt: serverTimestamp()
      });

      alert("上传成功！新发型已添加到推荐列表。");
      onBack();
    } catch (error) {
      console.error("Upload failed", error);
      alert("上传失败，请重试");
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="p-5 space-y-6 h-full overflow-y-auto bg-slate-50">
      <div className="flex items-center gap-2 mb-4">
        <button onClick={onBack} className="p-2 bg-white rounded-lg border border-slate-200"><Undo2 size={20}/></button>
        <h2 className="text-xl font-bold text-slate-900">管理员：添加新发型</h2>
      </div>

      <div className="space-y-4">
        {/* Name & Desc */}
        <input 
          className="w-full p-3 border rounded-xl" 
          placeholder="发型名称 (如: 渣男锡纸烫)" 
          value={newStyle.name}
          onChange={e => setNewStyle({...newStyle, name: e.target.value})}
        />
        <input 
          className="w-full p-3 border rounded-xl" 
          placeholder="简短描述 (如: 适合圆脸，需要打理)" 
          value={newStyle.desc}
          onChange={e => setNewStyle({...newStyle, desc: e.target.value})}
        />
        <div className="flex items-center gap-3">
          <label className="text-sm font-bold text-slate-600">最低长度(cm):</label>
          <input 
            type="number" 
            className="w-20 p-2 border rounded-xl text-center" 
            value={newStyle.minLengthCm}
            onChange={e => setNewStyle({...newStyle, minLengthCm: parseInt(e.target.value) || 0})}
          />
        </div>

        {/* Image Upload */}
        <div className="border-2 border-dashed border-slate-300 rounded-xl p-4 flex flex-col items-center justify-center bg-white">
          {imagePreview ? (
            <div className="relative w-full h-48">
              <img src={imagePreview} className="w-full h-full object-cover rounded-lg" alt="preview" />
              <button onClick={() => {setImageFile(null); setImagePreview(null)}} className="absolute top-2 right-2 bg-red-500 text-white p-1 rounded-full"><X size={16}/></button>
            </div>
          ) : (
            <label className="flex flex-col items-center gap-2 cursor-pointer w-full py-8">
              <UploadCloud size={40} className="text-slate-400"/>
              <span className="text-sm text-slate-500">点击上传参考图</span>
              <input type="file" accept="image/*" className="hidden" onChange={handleFileChange} />
            </label>
          )}
        </div>

        {/* Config Presets */}
        <div className="space-y-4 pt-4 border-t">
          <h3 className="font-bold text-slate-700">关联参数配置</h3>
          
          <div className="grid grid-cols-2 gap-2">
             <div className="text-xs font-bold text-slate-500">两侧设定</div>
             <select className="border p-2 rounded text-sm w-full" onChange={(e) => handleConfigSelect('sides', OPTIONS.sides.find(i=>i.id===e.target.value))}>
                {OPTIONS.sides.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
             </select>
          </div>
          <div className="grid grid-cols-2 gap-2">
             <div className="text-xs font-bold text-slate-500">头顶设定</div>
             <select className="border p-2 rounded text-sm w-full" onChange={(e) => handleConfigSelect('top', OPTIONS.top.find(i=>i.id===e.target.value))}>
                {OPTIONS.top.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
             </select>
          </div>
          <div className="grid grid-cols-2 gap-2">
             <div className="text-xs font-bold text-slate-500">刘海设定</div>
             <select className="border p-2 rounded text-sm w-full" onChange={(e) => handleConfigSelect('bangs', OPTIONS.bangs.find(i=>i.id===e.target.value))}>
                {OPTIONS.bangs.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
             </select>
          </div>
          <div className="grid grid-cols-2 gap-2">
             <div className="text-xs font-bold text-slate-500">后颈设定</div>
             <select className="border p-2 rounded text-sm w-full" onChange={(e) => handleConfigSelect('back', OPTIONS.back.find(i=>i.id===e.target.value))}>
                {OPTIONS.back.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
             </select>
          </div>
        </div>

        <button 
          onClick={handleSubmit}
          disabled={uploading}
          className="w-full py-4 bg-slate-900 text-white rounded-xl font-bold flex items-center justify-center gap-2 disabled:opacity-50"
        >
          {uploading ? <Loader2 className="animate-spin" /> : <Save size={20} />}
          {uploading ? '上传中...' : '保存并发布'}
        </button>
      </div>
    </div>
  );
};

export default function HairCutTalk() {
  const [step, setStep] = useState(0);
  const [appMode, setAppMode] = useState('edit'); 
  const [user, setUser] = useState(null);
  const [showAdmin, setShowAdmin] = useState(false);
  const [dynamicStyles, setDynamicStyles] = useState(DEFAULT_STYLES);
  
  const fileInputRef = useRef(null);
  const [userData, setUserData] = useState({ currentLengthLevel: 2, photo: null });
  const [preferences, setPreferences] = useState({
    selectedStyleId: null, growthTime: 0,
    sides: null, top: null, bangs: null, back: null,
    customDonts: [], notes: '', lengthChange: 1, lengthMode: 'cut', 
  });

  const totalSteps = 6; 

  // --- Auth & Data Fetching ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  useEffect(() => {
    if (!user) return;
    const fetchStyles = async () => {
       try {
         const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'styles'), orderBy('createdAt', 'desc'));
         const snapshot = await getDocs(q);
         const fetched = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
         setDynamicStyles([...fetched, ...DEFAULT_STYLES]); // Newest first
       } catch (e) {
         console.error("Fetch styles error", e);
       }
    };
    fetchStyles();
  }, [user, showAdmin]); // Re-fetch when exiting admin mode to see updates

  // --- Actions ---
  const handlePhotoUpload = (event) => {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => setUserData(prev => ({ ...prev, photo: reader.result }));
      reader.readAsDataURL(file);
    }
  };

  const applyPreset = (style) => {
    const months = calculateGrowthTime(userData.currentLengthLevel, style.minLengthCm);
    setPreferences(prev => ({
      ...prev, selectedStyleId: style.id, growthTime: months,
      sides: style.config.sides, top: style.config.top, bangs: style.config.bangs, back: style.config.back,
    }));
    nextStep();
  };

  const handleSelect = (category, value) => setPreferences(prev => ({ ...prev, [category]: value }));
  const toggleDont = (item) => setPreferences(prev => ({
    ...prev, customDonts: prev.customDonts.includes(item) ? prev.customDonts.filter(i => i !== item) : [...prev.customDonts, item]
  }));
  const nextStep = () => setStep(prev => Math.min(prev + 1, totalSteps - 1));
  const prevStep = () => setStep(prev => Math.max(prev - 1, 0));
  const reset = () => { if(window.confirm("重置所有选项？")) { setStep(0); setAppMode('edit'); setUserData({currentLengthLevel:2,photo:null}); setPreferences({selectedStyleId:null,growthTime:0,sides:null,top:null,bangs:null,back:null,customDonts:[],notes:'',lengthChange:1,lengthMode:'cut'}); }};

  // --- Screens ---

  const WelcomeScreen = () => (
    <div className="flex flex-col items-center justify-center h-full p-6 text-center space-y-8 bg-slate-900 text-white relative overflow-hidden">
      <div className="absolute top-0 left-0 w-full h-full opacity-10 pointer-events-none">
         <div className="absolute top-10 left-[-20px] text-9xl">✂️</div>
         <div className="absolute bottom-20 right-[-20px] text-9xl">💈</div>
      </div>
      <div className="w-24 h-24 bg-blue-600 rounded-2xl flex items-center justify-center shadow-[0_0_30px_rgba(37,99,235,0.5)] z-10">
        <Scissors size={48} className="text-white" />
      </div>
      <div className="z-10">
        <h1 className="text-3xl font-black mb-3 tracking-tight">男士理发<br/>沟通神器</h1>
        <p className="text-slate-400 max-w-xs mx-auto text-sm leading-relaxed">
          精准选择，拒绝翻车。<br/>
          包含<span className="text-white font-bold">留发时长预估</span><br/>
          及 <span className="text-pink-400 font-bold">女友审批模式</span>。
        </p>
      </div>
      <div className="z-10 w-full max-w-xs space-y-4">
        <button 
          onClick={nextStep}
          className="w-full py-4 bg-white text-slate-900 rounded-xl font-bold text-lg active:scale-95 transition-transform flex items-center justify-center gap-2 shadow-xl"
        >
          开始定制 <ChevronRight />
        </button>
        <button 
          onClick={() => setShowAdmin(true)}
          className="w-full py-2 text-slate-600 text-xs font-medium hover:text-white transition-colors flex items-center justify-center gap-1"
        >
          <UploadCloud size={14}/> 我是发型师 (上传参考图)
        </button>
      </div>
    </div>
  );

  const CurrentStateScreen = () => (
    <div className="p-5 space-y-8">
      <div className="space-y-2">
        <h2 className="text-2xl font-black text-slate-900">1. 你的现状</h2>
        <p className="text-slate-500 text-sm">我们会帮你计算还需要留多久头发。</p>
      </div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-700 uppercase">当前头发长度</label>
        <div className="grid grid-cols-2 gap-3">
          {CURRENT_LENGTHS.map((len) => (
            <button key={len.id} onClick={() => setUserData(p => ({ ...p, currentLengthLevel: len.level }))} className={`p-3 rounded-xl border-2 text-left transition-all ${userData.currentLengthLevel === len.level ? 'border-blue-600 bg-blue-50 ring-1 ring-blue-600' : 'border-slate-200 bg-white hover:border-slate-300'}`}>
              <div className="font-bold text-slate-800">{len.label}</div>
              <div className="text-xs text-slate-500">{len.cm}</div>
            </button>
          ))}
        </div>
      </div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-700 uppercase flex justify-between">
           <span>上传正脸照片 (可选)</span>
           {userData.photo && <span className="text-green-600 text-xs flex items-center gap-1"><Check size={12}/> 已上传</span>}
        </label>
        <div onClick={() => fileInputRef.current?.click()} className="relative aspect-video w-full max-w-[240px] mx-auto rounded-2xl border-2 border-dashed border-slate-300 bg-slate-100 flex flex-col items-center justify-center cursor-pointer hover:bg-slate-200 transition-colors overflow-hidden group">
          {userData.photo ? <img src={userData.photo} alt="User" className="w-full h-full object-cover" /> : <><Camera size={32} className="text-slate-400 mb-2 group-hover:text-slate-600" /><span className="text-xs text-slate-500 font-medium">点击上传 / 拍摄</span></>}
          <input type="file" ref={fileInputRef} onChange={handlePhotoUpload} accept="image/*" className="hidden" />
        </div>
      </div>
    </div>
  );

  const RecommendationScreen = () => {
    return (
      <div className="p-5 space-y-6">
        <div className="space-y-2">
          <h2 className="text-2xl font-black text-slate-900">2. 目标风格</h2>
          <p className="text-slate-500 text-sm">包含系统预设及最新上传的款式。</p>
        </div>
        <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
          {dynamicStyles.map(style => {
            const monthsToGrow = calculateGrowthTime(userData.currentLengthLevel, style.minLengthCm);
            const isReady = monthsToGrow === 0;
            return (
              <div key={style.id} onClick={() => applyPreset(style)} className="group cursor-pointer bg-white rounded-2xl border border-slate-200 shadow-sm overflow-hidden hover:shadow-md hover:border-blue-400 transition-all relative">
                <div className="h-40 overflow-hidden relative">
                   <img src={style.image} alt={style.name} className={`w-full h-full object-cover transition-transform duration-500 group-hover:scale-110 ${!isReady ? 'grayscale-[50%]' : ''}`} />
                   <div className="absolute inset-0 bg-gradient-to-t from-black/70 to-transparent flex items-end p-4">
                      <div>
                        <span className="text-white font-bold text-lg block">{style.name}</span>
                        <div className={`inline-flex items-center gap-1 px-2 py-0.5 rounded text-[10px] font-bold mt-1 ${isReady ? 'bg-green-500 text-white' : 'bg-yellow-500 text-slate-900'}`}>
                           {isReady ? <><Check size={10} /> 长度达标</> : <><Clock size={10} /> 需留 {monthsToGrow} 个月</>}
                        </div>
                      </div>
                   </div>
                </div>
                <div className="p-4">
                  <p className="text-xs text-slate-500 mb-3">{style.desc}</p>
                  <div className="flex gap-2 text-[10px] text-slate-600">
                    <span className="bg-slate-100 px-2 py-1 rounded">{style.config.sides.label}</span>
                    <span className="bg-slate-100 px-2 py-1 rounded">{style.config.bangs.label}</span>
                  </div>
                </div>
              </div>
            );
          })}
          <button onClick={() => applyPreset({ id: 'custom', name: '完全自定义', minLengthCm: 0, config: { sides: OPTIONS.sides[0], top: OPTIONS.top[0], bangs: OPTIONS.bangs[0], back: OPTIONS.back[0] } })} className="w-full py-4 text-sm text-slate-500 font-medium border-2 border-dashed border-slate-300 rounded-xl hover:border-slate-400 hover:text-slate-700">
            跳过推荐，我自己设置参数
          </button>
        </div>
      </div>
    );
  };

  const ZoneSelection = () => (
    <div className="p-5 space-y-8">
      <div className="flex items-center justify-between"><h2 className="text-xl font-bold text-slate-900">3. 侧/后 细节</h2></div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-500 uppercase flex items-center gap-2">两侧 Sides</label>
        <div className="grid grid-cols-2 gap-3">{OPTIONS.sides.map(opt => <ImageOptionCard key={opt.id} item={opt} selected={preferences.sides?.id === opt.id} onClick={() => handleSelect('sides', opt)} />)}</div>
      </div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-500 uppercase flex items-center gap-2">后颈 Back</label>
        <div className="grid grid-cols-3 gap-2">{OPTIONS.back.map(opt => <ImageOptionCard key={opt.id} item={opt} small selected={preferences.back?.id === opt.id} onClick={() => handleSelect('back', opt)} />)}</div>
      </div>
    </div>
  );

  const TopSelection = () => (
    <div className="p-5 space-y-8">
      <div className="flex items-center justify-between"><h2 className="text-xl font-bold text-slate-900">4. 顶/前 细节</h2></div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-500 uppercase">头顶 Top</label>
        <div className="grid grid-cols-2 gap-3">{OPTIONS.top.map(opt => <ImageOptionCard key={opt.id} item={opt} selected={preferences.top?.id === opt.id} onClick={() => handleSelect('top', opt)} />)}</div>
      </div>
      <div className="space-y-3">
        <label className="text-sm font-bold text-slate-500 uppercase">刘海 Bangs</label>
        <div className="grid grid-cols-2 gap-3">{OPTIONS.bangs.map(opt => <ImageOptionCard key={opt.id} item={opt} selected={preferences.bangs?.id === opt.id} onClick={() => handleSelect('bangs', opt)} />)}</div>
      </div>
      <div className="bg-white p-4 rounded-xl border border-slate-200 space-y-4 shadow-sm">
         <div className="flex justify-between items-center">
            <label className="text-sm font-bold text-slate-700 flex items-center gap-2"><Ruler size={16}/> 整体长度微调</label>
            <div className="bg-slate-100 rounded-lg p-0.5 flex text-xs">
              <button onClick={() => setPreferences(p => ({...p, lengthMode: 'cut'}))} className={`px-3 py-1.5 rounded-md transition-all ${preferences.lengthMode === 'cut' ? 'bg-white text-blue-700 shadow-sm font-bold' : 'text-slate-500'}`}>剪掉</button>
              <button onClick={() => setPreferences(p => ({...p, lengthMode: 'retain'}))} className={`px-3 py-1.5 rounded-md transition-all ${preferences.lengthMode === 'retain' ? 'bg-white text-blue-700 shadow-sm font-bold' : 'text-slate-500'}`}>保留</button>
            </div>
         </div>
         <div className="flex items-center gap-4">
           <input type="range" min="0.5" max="5" step="0.5" value={preferences.lengthChange} onChange={(e) => setPreferences({...preferences, lengthChange: e.target.value})} className="flex-1 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-blue-600" />
           <div className="bg-blue-50 text-blue-700 px-2 py-1 rounded font-mono font-bold w-16 text-center">{preferences.lengthChange}cm</div>
         </div>
      </div>
    </div>
  );

  const ApprovalScreen = () => (
    <div className="h-full bg-pink-50 flex flex-col items-center pt-10 px-6 relative">
      <div className="absolute top-0 left-0 w-full h-2 bg-gradient-to-r from-pink-300 to-purple-400"></div>
      <div className="bg-white p-4 rounded-full shadow-xl mb-6 ring-4 ring-pink-100"><Heart size={40} className="text-pink-500 fill-pink-500 animate-pulse" /></div>
      <h2 className="text-2xl font-black text-slate-800 mb-2">女友审批模式</h2>
      <p className="text-slate-500 text-center mb-8">请女朋友审核该发型方案。<br/>是否同意他剪成这样？</p>
      <div className="w-full bg-white rounded-2xl shadow-xl overflow-hidden mb-8 border border-pink-100">
         <div className="h-48 bg-slate-200 relative">
            {userData.photo ? <img src={userData.photo} className="w-full h-full object-cover" alt="Boy" /> : <div className="w-full h-full flex items-center justify-center bg-slate-100 text-slate-300"><User size={64} /></div>}
            <div className="absolute bottom-0 left-0 w-full bg-gradient-to-t from-black/80 to-transparent p-4 text-white">
               <div className="font-bold text-lg">{preferences.selectedStyleId ? dynamicStyles.find(s=>s.id===preferences.selectedStyleId)?.name : '自定义发型'}</div>
               {preferences.growthTime > 0 && <div className="text-yellow-300 text-xs font-bold mt-1 flex items-center gap-1"><Clock size={12}/> 需要留发约 {preferences.growthTime} 个月</div>}
            </div>
         </div>
         <div className="p-4 grid grid-cols-2 gap-2 text-xs">
            <div className="bg-slate-50 p-2 rounded"><span className="text-slate-400 block mb-1">两侧</span><span className="font-bold text-slate-700">{preferences.sides?.label}</span></div>
            <div className="bg-slate-50 p-2 rounded"><span className="text-slate-400 block mb-1">刘海</span><span className="font-bold text-slate-700">{preferences.bangs?.label}</span></div>
            {preferences.customDonts.length > 0 && <div className="col-span-2 bg-red-50 p-2 rounded text-red-600 border border-red-100"><span className="font-bold block mb-1 flex items-center gap-1"><AlertTriangle size={10}/> 避雷重点</span>{preferences.customDonts.join(', ')}</div>}
         </div>
      </div>
      <div className="w-full flex gap-4 mt-auto mb-8">
         <button onClick={() => setAppMode('edit')} className="flex-1 py-4 bg-white border-2 border-slate-200 text-slate-500 rounded-2xl font-bold flex flex-col items-center justify-center gap-1 active:scale-95 transition-transform"><ThumbsDown size={24} />驳回修改</button>
         <button onClick={() => setAppMode('approved')} className="flex-1 py-4 bg-gradient-to-br from-pink-500 to-purple-600 text-white rounded-2xl font-bold flex flex-col items-center justify-center gap-1 shadow-lg shadow-pink-200 active:scale-95 transition-transform"><Heart size={24} className="fill-white"/>批准通过</button>
      </div>
    </div>
  );

  const SummaryCard = () => (
    <div className="h-full bg-slate-900 p-4 flex flex-col text-white">
      <div className="flex-1 bg-white text-slate-900 rounded-2xl overflow-hidden flex flex-col shadow-2xl relative">
        <div className="bg-blue-600 p-4 text-white flex justify-between items-start">
           <div>
             <h2 className="font-black text-2xl flex items-center gap-2">理发需求单 <Scissors size={20} className="text-blue-200"/></h2>
             {appMode === 'approved' && <span className="inline-block bg-pink-500 text-white text-[10px] px-2 py-0.5 rounded-full font-bold mt-1 shadow-sm">❤️ 女友已批准</span>}
             {appMode === 'self_approved' && <span className="inline-block bg-slate-800 text-yellow-400 border border-yellow-400 text-[10px] px-2 py-0.5 rounded-full font-bold mt-1 shadow-sm flex items-center gap-1 w-fit"><Crown size={10} fill="currentColor"/> 本人已确认</span>}
           </div>
           {userData.photo ? <div className="w-16 h-16 rounded-lg border-2 border-white overflow-hidden bg-slate-200"><img src={userData.photo} className="w-full h-full object-cover" alt="user" /></div> : <div className="w-16 h-16 rounded-lg border-2 border-white/30 flex items-center justify-center bg-blue-700"><User className="text-blue-300" /></div>}
        </div>
        <div className="flex-1 overflow-y-auto p-4 space-y-5">
           {preferences.growthTime > 0 && <div className="bg-yellow-50 border border-yellow-200 p-3 rounded-xl flex items-start gap-3"><Clock className="text-yellow-600 mt-0.5" size={20} /><div><h4 className="font-bold text-yellow-800 text-sm">养成计划中</h4><p className="text-yellow-700 text-xs mt-1">此发型需要更长的头发。预计还需留发 <span className="font-bold text-lg">{preferences.growthTime}</span> 个月。<br/>本次理发请以 <span className="font-bold">“留长过渡，只修轮廓”</span> 为主。</p></div></div>}
           <div><div className="flex justify-between items-center mb-2"><h3 className="text-xs font-bold text-slate-400 uppercase flex items-center gap-1"><AlertTriangle size={12}/> 避雷区</h3></div><div className="flex flex-wrap gap-2">{preferences.customDonts.length > 0 ? preferences.customDonts.map((d, i) => <span key={i} className="px-2 py-1 text-xs rounded border bg-red-500 text-white border-red-500 font-bold">{d}</span>) : <span className="text-xs text-slate-400">无特殊避雷</span>}</div></div>
           <div className="grid grid-cols-2 gap-3">
              {[{ label: '两侧', obj: preferences.sides }, { label: '头顶', obj: preferences.top }, { label: '刘海', obj: preferences.bangs }, { label: '后颈', obj: preferences.back }].map((part, idx) => (
                <div key={idx} className="bg-slate-50 rounded-lg border border-slate-100 overflow-hidden flex flex-col">
                   <div className="h-16 bg-slate-200 relative">
                     {part.obj?.image ? <img src={part.obj.image} className="w-full h-full object-cover opacity-80" alt="" /> : <div className="w-full h-full flex items-center justify-center text-slate-400"><ImageIcon size={16}/></div>}
                     <div className="absolute top-0 left-0 bg-blue-600 text-white text-[10px] px-1.5 py-0.5 rounded-br font-bold">{part.label}</div>
                   </div>
                   <div className="p-2"><div className="font-bold text-sm text-slate-800 line-clamp-1">{part.obj?.label || '未指定'}</div></div>
                </div>
              ))}
           </div>
           <div className="bg-slate-50 p-3 rounded-lg border border-slate-200 text-sm"><div className="flex items-center gap-2 text-slate-800 font-bold mb-1"><Ruler size={14}/> 长度控制</div><div className="text-slate-600">整体{preferences.lengthMode === 'cut' ? '剪去' : '保留'} <span className="font-bold text-lg text-slate-900">{preferences.lengthChange}cm</span> 左右</div></div>
           {preferences.notes && <div><h3 className="text-xs font-bold text-slate-400 uppercase mb-1">特殊备注</h3><div className="bg-yellow-50 p-3 rounded-lg border border-yellow-100 text-slate-800 text-sm">{preferences.notes}</div></div>}
        </div>
      </div>
      <div className="mt-4 flex-shrink-0">
         {(appMode === 'approved' || appMode === 'self_approved') ? (
           <div className="flex gap-3">
              <button onClick={() => setAppMode('edit')} className="px-4 py-3 rounded-xl bg-slate-800 text-slate-400 font-bold">修改</button>
              <button onClick={() => alert("请截屏保存此页面")} className="flex-1 py-3 rounded-xl bg-green-500 text-white font-bold flex items-center justify-center gap-2 shadow-lg active:scale-95 transition-transform"><Save size={18}/> 保存最终卡片</button>
           </div>
         ) : (
           <div className="flex flex-col gap-2">
              <button onClick={() => setAppMode('pending_approval')} className="w-full py-3 rounded-xl bg-pink-500 text-white font-bold flex items-center justify-center gap-2 shadow-lg active:scale-95 transition-transform"><Lock size={18}/> 提交女友审批 (保命要紧)</button>
              <div className="flex gap-3">
                <button onClick={prevStep} className="px-4 py-3 rounded-xl bg-slate-800 text-slate-400 font-bold">返回</button>
                <button onClick={() => setAppMode('self_approved')} className="flex-1 py-3 rounded-xl bg-blue-600 text-white font-bold flex items-center justify-center gap-2 active:scale-95 transition-transform"><Crown size={18}/> 我自己做主 (单身版)</button>
              </div>
           </div>
         )}
      </div>
    </div>
  );

  const FooterNav = () => {
    if (step === 0 || step === totalSteps - 1) return null;
    return (
      <div className="fixed bottom-0 left-0 right-0 max-w-md mx-auto p-4 bg-white border-t border-slate-200 flex justify-between items-center z-50 safe-area-bottom">
        <button onClick={prevStep} className="px-4 py-2 text-slate-500 font-medium hover:bg-slate-50 rounded-lg flex items-center gap-1"><Undo2 size={16}/> 上一步</button>
        <div className="flex gap-1">{[...Array(totalSteps-2)].map((_, i) => <div key={i} className={`h-1.5 rounded-full transition-all ${i + 1 === step ? 'w-6 bg-blue-600' : 'w-2 bg-slate-200'}`} />)}</div>
        <button onClick={nextStep} className={`px-6 py-2 bg-slate-900 text-white rounded-lg font-semibold shadow-lg hover:bg-slate-800 flex items-center gap-2`}>{step === totalSteps - 2 ? '生成预览' : '下一步'} <ChevronRight size={16} /></button>
      </div>
    );
  };

  if (showAdmin) return <AdminUploadScreen onBack={() => setShowAdmin(false)} user={user} />;
  if (appMode === 'pending_approval') return <div className="max-w-md mx-auto h-screen overflow-hidden shadow-2xl sm:border-x border-slate-200 font-sans"><ApprovalScreen /></div>;

  return (
    <div className="max-w-md mx-auto h-screen bg-slate-50 flex flex-col overflow-hidden shadow-2xl sm:border-x border-slate-200 font-sans relative">
      {step > 0 && step < totalSteps - 1 && <div className="bg-white px-4 py-3 border-b border-slate-100 flex justify-between items-center z-50"><h1 className="font-bold text-slate-800 flex items-center gap-2"><Scissors size={20} className="text-blue-600"/> 发型定制</h1><button onClick={reset} className="text-slate-400 hover:text-red-500 transition-colors"><RefreshCw size={18} /></button></div>}
      <StepWizard currentStep={step}><WelcomeScreen /><CurrentStateScreen /><RecommendationScreen /><ZoneSelection /><TopSelection /><SummaryCard /></StepWizard>
      <FooterNav />
    </div>
  );
