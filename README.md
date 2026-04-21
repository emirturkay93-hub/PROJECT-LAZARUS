-- ============================================================
-- [[ MERT SCRIPT - PROJECT DELTA V21 ]]
-- [[ by Mert | Player+NPC ESP + Kaçış ESP + Aimbot + FOV ]]
-- ============================================================

-- ── CLEANUP ─────────────────────────────────────────────────
if getgenv()._MertRunning then getgenv()._MertRunning = false; task.wait(0.3) end
getgenv()._MertRunning = true
if getgenv()._MertConns then
    for _, c in pairs(getgenv()._MertConns) do pcall(function() c:Disconnect() end) end
end
getgenv()._MertConns = {}
if getgenv()._MertDrawings then
    for _, d in pairs(getgenv()._MertDrawings) do pcall(function() d:Remove() end) end
end
getgenv()._MertDrawings = {}

-- ── SERVICES ────────────────────────────────────────────────
local Players  = game:GetService("Players")
local RunSvc   = game:GetService("RunService")
local UIS      = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local WS       = game:GetService("Workspace")
local Camera   = WS.CurrentCamera
local LP       = Players.LocalPlayer
local OrigFOV  = Camera.FieldOfView

-- ── CONFIG ──────────────────────────────────────────────────
local CFG = {
    ESP = {
        Enabled  = true,
        Players  = true,
        NPCs     = true,
        Escape   = true,
        Chams    = true,
        Names    = true,   -- İsim göster/gizle
        Distance = true,
        Weapon   = true,
        BoxESP   = true,   -- KESİNLİKLE SİLME: Kutu ESP
        MaxDist  = 2000,
    },
    Aimbot = {
        Enabled  = false,
        ShowFOV  = true,
        FOV      = 150,
        Part     = "Head",
        Aiming   = false,
        Predict  = false,
        TargetNPC = true,  -- Hem oyuncuları hem NPC'leri hedef al
    },
    World = {
        AlwaysDay  = false,
        Fullbright = false,
        CustomFOV  = false,
        FOVValue   = 100,
    },
    Perf = {
        LowGFX      = false,  -- Grafik kalitesini duşur
        NoShadows   = false,  -- Gölgeleri kapat
        NoParticles = false,  -- Partikülleri kaldır
        NoFog       = false,  -- Sisi kaldır
        FPSCap      = 0,      -- 0 = sınırsız
    },
}
getgenv()._MertCFG = CFG

-- ── DRAWING HELPERS ─────────────────────────────────────────
local function Sq(filled, color, trans)
    local s = Drawing.new("Square")
    s.Filled = filled; s.Color = color or Color3.new(0,0,0)
    s.Transparency = trans or 1; s.Visible = false
    if not filled then s.Thickness = 1 end
    table.insert(getgenv()._MertDrawings, s); return s
end
local function Tx(size, color, center)
    local t = Drawing.new("Text")
    t.Font = Drawing.Fonts.UI; t.Size = size or 13
    t.Outline = true; t.OutlineColor = Color3.new(0,0,0)
    t.Center = (center ~= false)
    t.Color = color or Color3.new(1,1,1); t.Visible = false
    table.insert(getgenv()._MertDrawings, t); return t
end
local function Ci(color, radius, filled, thick)
    local c = Drawing.new("Circle")
    c.Color = color or Color3.new(1,1,1); c.Radius = radius or 10
    c.Filled = filled or false; c.Thickness = thick or 1.5
    c.Visible = false; table.insert(getgenv()._MertDrawings, c); return c
end
local function Li(color, thick)  -- KESİNLİKLE SİLME: Box ESP için gerekli
    local l = Drawing.new("Line")
    l.Color = color or Color3.new(1,1,1); l.Thickness = thick or 1.5
    l.Visible = false; table.insert(getgenv()._MertDrawings, l); return l
end
local function Hide(t) for _, v in pairs(t) do v.Visible = false end end

-- ── FOV CIRCLE ──────────────────────────────────────────────
local FovCircle = Ci(Color3.fromRGB(220,220,220), 150, false, 1.5)

-- ── SAĞ ÜST HEDEF PANELİ ────────────────────────────────────
local Panel = {
    bg     = Sq(true,  Color3.fromRGB(10,10,15),  0.6),
    border = Sq(false, Color3.fromRGB(60,60,80),   1),
    topbar = Sq(true,  Color3.fromRGB(60,120,255), 1),
    title  = Tx(11, Color3.fromRGB(140,180,255), false),
    name   = Tx(14, Color3.new(1,1,1), false),
    info   = Tx(11, Color3.fromRGB(200,200,200), false),
    hpBg   = Sq(true, Color3.fromRGB(30,10,10), 1),
    hpFill = Sq(true, Color3.fromRGB(80,255,100), 1),
    hpTx   = Tx(10, Color3.new(1,1,1), false),
}
local function HidePanel() Hide(Panel) end

-- ── ESP KART POOL ────────────────────────────────────────────
local CardPool = {}
local function GetCard(key)
    if not CardPool[key] then
        CardPool[key] = {
            bg     = Sq(true,  Color3.fromRGB(8,8,12),   0.60),
            border = Sq(false, Color3.fromRGB(55,55,75),  1),
            topbar = Sq(true,  Color3.fromRGB(60,120,255),1),
            name   = Tx(13, Color3.new(1,1,1)),
            sub    = Tx(11, Color3.fromRGB(190,190,190)),
            hpBg   = Sq(true, Color3.fromRGB(20,5,5), 1),
            hpFill = Sq(true, Color3.fromRGB(80,255,100), 1),
            hpTx   = Tx(10, Color3.new(1,1,1)),
            -- KESİNLİKLE SİLME: Box ESP çizgileri
            boxT   = Li(Color3.fromRGB(60,150,255), 1.5),
            boxB   = Li(Color3.fromRGB(60,150,255), 1.5),
            boxL   = Li(Color3.fromRGB(60,150,255), 1.5),
            boxR   = Li(Color3.fromRGB(60,150,255), 1.5),
        }
    end
    return CardPool[key]
end

-- ── FROM ESP.LUA: PREMİUM BİLGİ KARTI (KESİNLİKLE SİLME) ──────────────
-- esp.lua'daki GetInfoCard sistemi. Oyuncular için ek yüzen kart.
local _pDrawings = {}
local function GetInfoCard(p)
    if not _pDrawings[p] then
        _pDrawings[p] = {
            bg      = Sq(true,  Color3.fromRGB(15,15,18), 0.70),
            border  = Sq(false, Color3.fromRGB(40,40,50),  1),
            topStr  = Sq(true,  Color3.fromRGB(220,50,50), 1),
            nameText= Tx(13, Color3.fromRGB(255,255,255)),
            wpnText = Tx(11, Color3.fromRGB(200,200,200)),
            hpText  = Tx(10, Color3.fromRGB(100,255,100)),
            hpBg    = Sq(true, Color3.fromRGB(40,20,20), 1),
            hpFill  = Sq(true, Color3.fromRGB(80,255,80), 1),
        }
    end
    return _pDrawings[p]
end
local function HideInfoCard(c) for _, d in pairs(c) do d.Visible = false end end

-- ── KAÇIŞ ZONE POOL ─────────────────────────────────────────
local EscPool = {}
local function GetEscDraw(i)
    if not EscPool[i] then
        EscPool[i] = {
            outer = Ci(Color3.fromRGB(0,255,100), 20, false, 2.5),
            inner = Ci(Color3.fromRGB(50,255,130), 6, true, 1),
            label = Tx(13, Color3.fromRGB(80,255,140)),
            dist  = Tx(11, Color3.fromRGB(180,255,200)),
        }
    end
    return EscPool[i]
end

-- ── CHAMS ───────────────────────────────────────────────────
local function AddHL(char)
    if not char or char:FindFirstChild("_MHL") then return end
    local h = Instance.new("Highlight"); h.Name = "_MHL"
    h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    h.FillColor = Color3.fromRGB(200,40,40); h.FillTransparency = 0.75
    h.OutlineColor = Color3.new(1,1,1); h.OutlineTransparency = 0; h.Parent = char
end
local function RemHL(char)
    if not char then return end
    local h = char:FindFirstChild("_MHL"); if h then h:Destroy() end
end
local function WatchP(p)
    if p == LP then return end
    if p.Character then task.defer(AddHL, p.Character) end
    table.insert(getgenv()._MertConns, p.CharacterAdded:Connect(function(c) task.defer(AddHL, c) end))
    table.insert(getgenv()._MertConns, p.CharacterRemoving:Connect(RemHL))
end
for _, p in pairs(Players:GetPlayers()) do WatchP(p) end
table.insert(getgenv()._MertConns, Players.PlayerAdded:Connect(WatchP))
-- Oyuncu ayrıldığında info card temizle (_pDrawings)
table.insert(getgenv()._MertConns, Players.PlayerRemoving:Connect(function(p)
    if _pDrawings[p] then
        for _, d in pairs(_pDrawings[p]) do pcall(function() d:Remove() end) end
        _pDrawings[p] = nil
    end
    if CardPool[p] then
        for _, d in pairs(CardPool[p]) do pcall(function() if d.Remove then d:Remove() end end) end
        CardPool[p] = nil
    end
end))

-- ── SİLAH TESPİTİ ───────────────────────────────────────────
local function GetWeapon(char)
    if not char then return "Yumruk" end
    local tool = char:FindFirstChildOfClass("Tool")
    if tool then return tool.Name end
    for _, c in pairs(char:GetChildren()) do
        if c:IsA("Model") then
            local n = c.Name:lower()
            if n ~= "armor" and n ~= "helmet" and n ~= "vest" then
                if c:FindFirstChild("Handle") or c:FindFirstChild("Muzzle") or
                   c:FindFirstChild("Receiver") or c:FindFirstChild("Barrel") then
                    return c.Name
                end
                for _, d in pairs(c:GetDescendants()) do
                    local pn = d.Name:lower()
                    if pn:find("muzzle") or pn:find("barrel") or pn:find("firepoint") then
                        return c.Name
                    end
                end
            end
        end
    end
    return "Yumruk"
end

-- ── NPC CACHE ───────────────────────────────────────────────
local npcCache, npcTimer = {}, 0
local function ScanNPCs()
    if tick() - npcTimer < 3 then return npcCache end
    npcTimer = tick(); npcCache = {}
    for _, v in pairs(WS:GetChildren()) do
        if v:IsA("Model") and v ~= LP.Character and not Players:GetPlayerFromCharacter(v) then
            if v:FindFirstChildOfClass("Humanoid") then
                table.insert(npcCache, v)
            end
        end
    end
    return npcCache
end

-- ── KAÇIŞ ZONE CACHE ────────────────────────────────────────
local EKWS = {"escape","exit","exfil","extract","extraction","exitzone","escapezone",
    "extractionzone","exitpoint","escapepoint","saferoom","evac","evacuation",
    "exitdoor","escapedoor","hatch","safe_exit","escape_zone","exit_point",
    "kaçış","kacis","çıkış","cikis","extract_zone"}
local escZones, escTimer = {}, 0
local function ScanEsc()
    if tick() - escTimer < 8 then return escZones end
    escTimer = tick(); escZones = {}
    for _, v in pairs(WS:GetDescendants()) do
        if v:IsA("BasePart") or v:IsA("Model") then
            local nl = v.Name:lower()
            for _, kw in pairs(EKWS) do
                if nl:find(kw, 1, true) then
                    local pos
                    if v:IsA("BasePart") then pos = v.Position
                    elseif v:IsA("Model") then
                        local pp = v.PrimaryPart or v:FindFirstChildOfClass("BasePart")
                        if pp then pos = pp.Position end
                    end
                    if pos then table.insert(escZones, {pos=pos, label=v.Name, obj=v}) end
                    break
                end
            end
        end
    end
    return escZones
end

-- ── FPS / PERFORMANS ────────────────────────────────────────
local _particlesRemoved = false
local function ApplyPerf()
    -- Grafik kalitesi
    pcall(function()
        if CFG.Perf.LowGFX then
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        else
            settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
        end
    end)
    -- Gölgeler
    if CFG.Perf.NoShadows then
        Lighting.GlobalShadows = false
    end
    -- Sis
    if CFG.Perf.NoFog then
        Lighting.FogEnd = 9e9; Lighting.FogStart = 9e9
    end
    -- Partiküller (bir kez kaldır)
    if CFG.Perf.NoParticles and not _particlesRemoved then
        _particlesRemoved = true
        for _, v in pairs(WS:GetDescendants()) do
            if v:IsA("ParticleEmitter") or v:IsA("Beam") or v:IsA("Trail") or v:IsA("Smoke") or v:IsA("Fire") or v:IsA("Sparkles") then
                pcall(function() v.Enabled = false end)
            end
        end
    end
    -- FPS cap
    pcall(function()
        if CFG.Perf.FPSCap > 0 then
            setfpscap(CFG.Perf.FPSCap)
        else
            setfpscap(0)
        end
    end)
end

local function GetTarget()
    local mPos = UIS:GetMouseLocation()
    local bestDist, bestPart = CFG.Aimbot.FOV, nil

    -- Oyuncular
    for _, p in pairs(Players:GetPlayers()) do
        if p == LP or not p.Character then continue end
        local part = p.Character:FindFirstChild(CFG.Aimbot.Part)
        local hum  = p.Character:FindFirstChildOfClass("Humanoid")
        if part and hum and hum.Health > 0 then
            local sp, vis = Camera:WorldToViewportPoint(part.Position)
            if vis then
                local d = (Vector2.new(sp.X, sp.Y) - mPos).Magnitude
                if d < bestDist then bestDist = d; bestPart = part end
            end
        end
    end

    -- NPC'ler (ayrı toggle ile)
    if CFG.Aimbot.TargetNPC then
        for _, ent in pairs(ScanNPCs()) do
            local part = ent:FindFirstChild(CFG.Aimbot.Part) or ent:FindFirstChild("HumanoidRootPart")
            local hum  = ent:FindFirstChildOfClass("Humanoid")
            if part and hum and hum.Health > 0 then
                local sp, vis = Camera:WorldToViewportPoint(part.Position)
                if vis then
                    local d = (Vector2.new(sp.X, sp.Y) - mPos).Magnitude
                    if d < bestDist then bestDist = d; bestPart = part end
                end
            end
        end
    end

    return bestPart
end

-- ── INPUT ───────────────────────────────────────────────────
table.insert(getgenv()._MertConns, UIS.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.UserInputType == Enum.UserInputType.MouseButton2 then CFG.Aimbot.Aiming = true end
end))
table.insert(getgenv()._MertConns, UIS.InputEnded:Connect(function(i, gp)
    if gp then return end
    if i.UserInputType == Enum.UserInputType.MouseButton2 then CFG.Aimbot.Aiming = false end
end))

-- ── YARDIMCI: KART ÇİZ ──────────────────────────────────────
local function DrawCard(card, cx, ty, w, h, nameStr, subStr, hp, hpMax, topClr, nameClr)
    card.bg.Color = Color3.fromRGB(8,8,12)
    card.bg.Size = Vector2.new(w,h); card.bg.Position = Vector2.new(cx-w/2,ty); card.bg.Visible = true
    card.border.Size = Vector2.new(w,h); card.border.Position = Vector2.new(cx-w/2,ty); card.border.Visible = true
    card.topbar.Color = topClr or Color3.fromRGB(60,120,255)
    card.topbar.Size = Vector2.new(w,2); card.topbar.Position = Vector2.new(cx-w/2,ty); card.topbar.Visible = true
    card.name.Color = nameClr or Color3.new(1,1,1)
    card.name.Text = nameStr; card.name.Position = Vector2.new(cx,ty+7); card.name.Visible = true
    if subStr and subStr ~= "" then
        card.sub.Text = subStr; card.sub.Position = Vector2.new(cx,ty+21); card.sub.Visible = true
    else card.sub.Visible = false end
    if hp and hpMax then
        local hpR = math.clamp(hp / math.max(hpMax,1), 0, 1)
        local hpC = hpR > 0.6 and Color3.fromRGB(60,245,90) or hpR > 0.3 and Color3.fromRGB(255,210,0) or Color3.fromRGB(255,55,55)
        card.hpBg.Size = Vector2.new(w,6); card.hpBg.Position = Vector2.new(cx-w/2,ty+h); card.hpBg.Visible = true
        card.hpFill.Color = hpC; card.hpFill.Size = Vector2.new(w*hpR,6)
        card.hpFill.Position = Vector2.new(cx-w/2,ty+h); card.hpFill.Visible = true
        card.hpTx.Text = math.floor(hp).."/"..math.floor(hpMax)
        card.hpTx.Color = hpC; card.hpTx.Position = Vector2.new(cx,ty+h+8); card.hpTx.Visible = true
    else
        card.hpBg.Visible = false; card.hpFill.Visible = false; card.hpTx.Visible = false
    end
end

-- ── RENDER LOOP ─────────────────────────────────────────────
table.insert(getgenv()._MertConns, RunSvc.RenderStepped:Connect(function()
    if not getgenv()._MertRunning then return end

    -- Dünya
    if CFG.World.AlwaysDay then Lighting.ClockTime = 14 end
    if CFG.World.Fullbright then
        Lighting.Ambient = Color3.new(1,1,1)
        Lighting.OutdoorAmbient = Color3.new(1,1,1)
        Lighting.GlobalShadows = false
    end
    -- Custom FOV: sadece değer değiştiğinde set et (her frame değil → lag düzeltildi)
    local _wantFOV = CFG.World.CustomFOV and CFG.World.FOVValue or OrigFOV
    if Camera.FieldOfView ~= _wantFOV then Camera.FieldOfView = _wantFOV end

    -- FOV dairesi
    local mPos = UIS:GetMouseLocation()
    if CFG.Aimbot.ShowFOV then
        FovCircle.Position = mPos; FovCircle.Radius = CFG.Aimbot.FOV; FovCircle.Visible = true
    else FovCircle.Visible = false end

    -- Aimbot
    local target = nil
    if CFG.Aimbot.Enabled or CFG.ESP.Enabled then
        target = GetTarget()
    end
    if CFG.Aimbot.Enabled and CFG.Aimbot.Aiming and target then
        local pos = target.Position
        if CFG.Aimbot.Predict then
            local root = target.Parent:FindFirstChild("HumanoidRootPart")
            if root then
                local v = root.AssemblyLinearVelocity
                pos = pos + v * ((Camera.CFrame.Position - pos).Magnitude / 1200)
            end
        end
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, pos)
    end

    -- ── SAĞ ÜST HEDEF PANELİ ────────────────────────────────
    if target and target.Parent then
        pcall(function()
            local fChar   = target.Parent
            local fPlayer = Players:GetPlayerFromCharacter(fChar)
            local fHum    = fChar:FindFirstChildOfClass("Humanoid")
            local fName   = fPlayer and fPlayer.Name or fChar.Name
            if fHum and fHum.Health > 0 then
                local scrW = Camera.ViewportSize.X
                local pW, pH = 230, 92
                local sx, sy = scrW - pW - 16, 16

                Panel.bg.Size = Vector2.new(pW,pH); Panel.bg.Position = Vector2.new(sx,sy); Panel.bg.Visible = true
                Panel.border.Size = Vector2.new(pW,pH); Panel.border.Position = Vector2.new(sx,sy); Panel.border.Visible = true
                Panel.topbar.Color = CFG.Aimbot.Aiming and Color3.fromRGB(255,60,60) or Color3.fromRGB(60,120,255)
                Panel.topbar.Size = Vector2.new(pW,3); Panel.topbar.Position = Vector2.new(sx,sy); Panel.topbar.Visible = true

                Panel.title.Text = fPlayer and "◈ OYUNCU HEDEFİ" or "◈ NPC HEDEFİ"
                Panel.title.Color = fPlayer and Color3.fromRGB(140,180,255) or Color3.fromRGB(255,220,100)
                Panel.title.Position = Vector2.new(sx+8,sy+7); Panel.title.Visible = true

                Panel.name.Text = fName
                Panel.name.Position = Vector2.new(sx+8,sy+23); Panel.name.Visible = true

                local wpn  = GetWeapon(fChar)
                local root = fChar:FindFirstChild("HumanoidRootPart") or target
                local dist = math.floor((Camera.CFrame.Position - root.Position).Magnitude)
                Panel.info.Text = "⚡ " .. dist .. "m   |   🔫 " .. wpn
                Panel.info.Position = Vector2.new(sx+8,sy+41); Panel.info.Visible = true

                local hpR = math.clamp(fHum.Health/math.max(fHum.MaxHealth,1),0,1)
                local hpC = hpR>0.6 and Color3.fromRGB(60,245,90) or hpR>0.3 and Color3.fromRGB(255,210,0) or Color3.fromRGB(255,55,55)
                Panel.hpBg.Size = Vector2.new(pW-16,8); Panel.hpBg.Position = Vector2.new(sx+8,sy+59); Panel.hpBg.Visible = true
                Panel.hpFill.Color = hpC; Panel.hpFill.Size = Vector2.new((pW-16)*hpR,8)
                Panel.hpFill.Position = Vector2.new(sx+8,sy+59); Panel.hpFill.Visible = true
                Panel.hpTx.Text = "CAN: "..math.floor(fHum.Health).." / "..math.floor(fHum.MaxHealth)
                Panel.hpTx.Color = hpC; Panel.hpTx.Position = Vector2.new(sx+8,sy+72); Panel.hpTx.Visible = true
            else HidePanel() end
        end)
    else HidePanel() end

    -- ── OYUNCU ESP ──────────────────────────────────────────
    for _, p in pairs(Players:GetPlayers()) do
        if p == LP then continue end
        local card = GetCard(p)
        local char = p.Character
        local head = char and char:FindFirstChild("Head")
        local root = char and (char:FindFirstChild("HumanoidRootPart") or head)
        local hum  = char and char:FindFirstChild("Humanoid")

        -- Chams güncelle
        if char then
            local hl = char:FindFirstChild("_MHL")
            if hl then hl.Enabled = CFG.ESP.Chams and CFG.ESP.Enabled end
        end

        -- Eski kart elemanlarını gizle (artık sadece box+infocard kullanıyoruz)
        card.bg.Visible=false; card.border.Visible=false; card.topbar.Visible=false
        card.name.Visible=false; card.sub.Visible=false
        card.hpBg.Visible=false; card.hpFill.Visible=false; card.hpTx.Visible=false
        if card.boxT then card.boxT.Visible=false; card.boxB.Visible=false; card.boxL.Visible=false; card.boxR.Visible=false end

        if not (CFG.ESP.Enabled and CFG.ESP.Players and char and head and root and hum and hum.Health > 0) then
            if _pDrawings[p] then HideInfoCard(_pDrawings[p]) end; continue
        end
        local sp, vis = Camera:WorldToViewportPoint(head.Position + Vector3.new(0,1.2,0))
        local dist = (Camera.CFrame.Position - root.Position).Magnitude
        if not (vis and sp.Z > 0 and dist <= CFG.ESP.MaxDist) then
            if _pDrawings[p] then HideInfoCard(_pDrawings[p]) end; continue
        end

        -- Sağlığı bir kere oku (race condition: 0 bug'ına karşı)
        local humHp  = math.max(hum.Health,  0)
        local humMax = math.max(hum.MaxHealth, 1)
        local isTarget = target and target.Parent == char

        -- BOX ESP (KESİNLİKLE SİLME)
        if CFG.ESP.BoxESP and card.boxT then
            local footSp = Camera:WorldToViewportPoint(root.Position - Vector3.new(0,3,0))
            local bH = math.abs(sp.Y - footSp.Y)
            local bW = math.max(30, bH * 0.55)
            local bx = sp.X - bW/2; local by = sp.Y - bH/2
            local bc = isTarget and Color3.fromRGB(255,80,80) or Color3.fromRGB(60,150,255)
            card.boxT.From=Vector2.new(bx,by);    card.boxT.To=Vector2.new(bx+bW,by);    card.boxT.Color=bc; card.boxT.Visible=true
            card.boxB.From=Vector2.new(bx,by+bH); card.boxB.To=Vector2.new(bx+bW,by+bH); card.boxB.Color=bc; card.boxB.Visible=true
            card.boxL.From=Vector2.new(bx,by);    card.boxL.To=Vector2.new(bx,by+bH);    card.boxL.Color=bc; card.boxL.Visible=true
            card.boxR.From=Vector2.new(bx+bW,by); card.boxR.To=Vector2.new(bx+bW,by+bH); card.boxR.Color=bc; card.boxR.Visible=true
        end

        -- PREMİUM BİLGİ KARTI — BAŞIN ÜSTÜNDE (KESİNLİKLE SİLME)
        local ic = GetInfoCard(p)
        local nameStr = p.Name
        if CFG.ESP.Distance then nameStr = nameStr .. " [" .. math.floor(dist) .. "m]" end
        local wpnNow = GetWeapon(char)  -- bir kere çağrılır, tekrar yok
        ic.nameText.Text = nameStr; ic.wpnText.Text = wpnNow
        local icW = math.max(120, math.max(
            ic.nameText.TextBounds.X > 0 and ic.nameText.TextBounds.X or #nameStr*7,
            ic.wpnText.TextBounds.X  > 0 and ic.wpnText.TextBounds.X  or #wpnNow*7
        ) + 22)
        local icH = 38; local icX = sp.X - icW/2; local icY = sp.Y - icH - 14

        local iTopC  = isTarget and Color3.fromRGB(255,50,50)  or Color3.fromRGB(100,150,255)
        local iNameC = isTarget and Color3.fromRGB(255,140,140) or Color3.fromRGB(255,255,255)

        ic.bg.Color = isTarget and Color3.fromRGB(28,8,8) or Color3.fromRGB(12,12,16)
        ic.bg.Size=Vector2.new(icW,icH); ic.bg.Position=Vector2.new(icX,icY); ic.bg.Visible=true
        ic.border.Size=Vector2.new(icW,icH); ic.border.Position=Vector2.new(icX,icY); ic.border.Visible=true
        ic.topStr.Color=iTopC; ic.topStr.Size=Vector2.new(icW,2); ic.topStr.Position=Vector2.new(icX,icY); ic.topStr.Visible=true

        -- İsim toggle (CFG.ESP.Names)
        if CFG.ESP.Names then
            ic.nameText.Color=iNameC; ic.nameText.Position=Vector2.new(sp.X,icY+5); ic.nameText.Visible=true
        else
            ic.nameText.Visible=false
        end

        -- Silah toggle (CFG.ESP.Weapon)
        if CFG.ESP.Weapon then
            ic.wpnText.Color = isTarget and Color3.fromRGB(255,210,80) or Color3.fromRGB(170,200,255)
            ic.wpnText.Position=Vector2.new(sp.X,icY+20); ic.wpnText.Visible=true
        else
            ic.wpnText.Visible=false
        end

        -- Can barı (humHp: bir kere okundu — 0 gösterme hatası giderildi)
        local hpR = math.clamp(humHp/humMax, 0, 1)
        local hpC2 = hpR>0.6 and Color3.fromRGB(80,255,80) or hpR>0.3 and Color3.fromRGB(255,200,0) or Color3.fromRGB(255,55,55)
        ic.hpBg.Size=Vector2.new(icW,5); ic.hpBg.Position=Vector2.new(icX,icY+icH); ic.hpBg.Visible=true
        ic.hpFill.Color=hpC2; ic.hpFill.Size=Vector2.new(icW*hpR,5); ic.hpFill.Position=Vector2.new(icX,icY+icH); ic.hpFill.Visible=true
        ic.hpText.Text=math.floor(humHp).." / "..math.floor(humMax).."hp"
        ic.hpText.Color=hpC2; ic.hpText.Position=Vector2.new(sp.X,icY+icH+7); ic.hpText.Visible=true
    end

    -- ── NPC ESP ──────────────────────────────────────────────
    if CFG.ESP.Enabled and CFG.ESP.NPCs then
        local currentNPCs = ScanNPCs()
        -- Hangi NPC'lerin aktif olduğunu takip et
        local activeSet = {}
        for _, ent in pairs(currentNPCs) do
            if ent and ent.Parent then activeSet[ent] = true end
        end
        -- Kart havuzundaki eski NPC kartlarını temizle
        for key, card in pairs(CardPool) do
            if type(key) ~= "userdata" then continue end
            -- Oyuncu olmayan, artık listede olmayan NPC kartları gizle
            if not Players:GetPlayerFromCharacter(key) and not activeSet[key] then
                Hide(card)
            end
        end
        -- Aktif NPC'lerin ESP'sini çiz
        for _, ent in pairs(currentNPCs) do
            pcall(function()
                -- Instance silinmişse hemen gizle
                if not ent or not ent.Parent then
                    if CardPool[ent] then Hide(CardPool[ent]) end
                    return
                end
                local card = GetCard(ent)
                local hum  = ent:FindFirstChildOfClass("Humanoid")
                local root = ent:FindFirstChild("HumanoidRootPart") or ent:FindFirstChild("Head")
                if not (hum and root and hum.Health > 0) then Hide(card); return end

                local sp, vis = Camera:WorldToViewportPoint(root.Position + Vector3.new(0,2,0))
                local dist = (Camera.CFrame.Position - root.Position).Magnitude
                if not (vis and sp.Z > 0 and dist <= CFG.ESP.MaxDist) then Hide(card); return end

                local nameStr = ent.Name
                if CFG.ESP.Distance then nameStr = nameStr .. "  [" .. math.floor(dist) .. "m]" end
                local wpnStr = CFG.ESP.Weapon and GetWeapon(ent) or ""
                local h = (wpnStr ~= "" and wpnStr ~= "Yumruk") and 40 or 27
                local w = math.max(110, (card.name.TextBounds.X > 0 and card.name.TextBounds.X or #nameStr*7) + 22)
                local isTarget = target and target.Parent == ent

                DrawCard(card, sp.X, sp.Y-h-10, w, h, nameStr,
                    (wpnStr ~= "" and wpnStr ~= "Yumruk") and wpnStr or nil,
                    hum.Health, hum.MaxHealth,
                    isTarget and Color3.fromRGB(255,120,0) or Color3.fromRGB(255,170,0),
                    Color3.fromRGB(255,240,100))
                -- NPC'lerde box ESP yok (kullanıcı isteği)
                if card.boxT then card.boxT.Visible=false; card.boxB.Visible=false; card.boxL.Visible=false; card.boxR.Visible=false end
            end)
        end
    else
        -- ESP kapalıysa tüm NPC kartlarını gizle
        for key, card in pairs(CardPool) do
            if type(key) == "userdata" and not Players:GetPlayerFromCharacter(key) then
                Hide(card)
            end
        end
    end

    -- ── KAÇIŞ ZONE ESP ──────────────────────────────────────
    if CFG.ESP.Enabled and CFG.ESP.Escape then
        for i, z in pairs(ScanEsc()) do
            pcall(function()
                local ed  = GetEscDraw(i)
                local pos = z.pos
                pcall(function()
                    if z.obj and z.obj.Parent then
                        if z.obj:IsA("BasePart") then pos = z.obj.Position
                        elseif z.obj:IsA("Model") then
                            local pp = z.obj.PrimaryPart or z.obj:FindFirstChildOfClass("BasePart")
                            if pp then pos = pp.Position end
                        end
                    end
                end)
                local sp, vis = Camera:WorldToViewportPoint(pos + Vector3.new(0,3,0))
                local dist = (Camera.CFrame.Position - pos).Magnitude
                if vis and sp.Z > 0 and dist <= CFG.ESP.MaxDist then
                    local p2 = Vector2.new(sp.X, sp.Y)
                    ed.outer.Position = p2; ed.outer.Visible = true
                    ed.inner.Position = p2; ed.inner.Visible = true
                    ed.label.Text = "✓ " .. z.label
                    ed.label.Position = Vector2.new(sp.X, sp.Y-32); ed.label.Visible = true
                    if CFG.ESP.Distance then
                        ed.dist.Text = math.floor(dist).."m"
                        ed.dist.Position = Vector2.new(sp.X, sp.Y+24); ed.dist.Visible = true
                    else ed.dist.Visible = false end
                else Hide(ed) end
            end)
        end
    else for _, ed in pairs(EscPool) do Hide(ed) end end
end))

-- ── RAYFIELD UI ─────────────────────────────────────────────
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()
local Win = Rayfield:CreateWindow({
    Name            = "Mert Script | Project Delta",
    LoadingTitle    = "Project Delta",
    LoadingSubtitle = "by Mert | V21",
    ConfigurationSaving = { Enabled = true, Folder = "MertScript", FileName = "PD_V21" },
    KeySystem = true,
    KeySettings = {
        Title    = "Project Delta — Key Doğrulama",
        Subtitle = "Geçerli bir key giriniz",
        Note     = "<b><font color=\"rgb(0,120,255)\">DEVELOPER</font></b>",
        FileName = "PD_V21_Key",
        SaveKey  = true,
        GrabKeyFromSite = false,
        Key      = {"MertDelta2025"},
        -- 24 saat sonra key süresi dolar
        KeyExpiry = 86400,   -- saniye cinsinden 24 saat
    },
})

-- GÖRSEL TAB
local ET = Win:CreateTab("👁 Görsel", 4483345998)

ET:CreateSection("Genel")
ET:CreateToggle({Name="ESP Aktif", CurrentValue=true, Flag="ESPEn",
    Callback=function(v) CFG.ESP.Enabled=v end})
ET:CreateToggle({Name="Mesafe Göster", CurrentValue=true, Flag="ESPDist",
    Callback=function(v) CFG.ESP.Distance=v end})

ET:CreateSection("Oyuncu ESP")
ET:CreateToggle({Name="Oyuncu ESP", CurrentValue=true, Flag="ESPPl",
    Callback=function(v) CFG.ESP.Players=v end})
ET:CreateToggle({Name="İsim Göster", CurrentValue=true, Flag="ESPName",
    Callback=function(v) CFG.ESP.Names=v end})
ET:CreateToggle({Name="Chams (Highlight)", CurrentValue=true, Flag="ESPCh",
    Callback=function(v) CFG.ESP.Chams=v end})
ET:CreateToggle({Name="Silah İsmi Göster", CurrentValue=true, Flag="ESPWpn",
    Callback=function(v) CFG.ESP.Weapon=v end})
ET:CreateToggle({Name="Kutu ESP (Box)", CurrentValue=true, Flag="ESPBox",  -- KESİNLİKLE SİLME
    Callback=function(v) CFG.ESP.BoxESP=v end})

ET:CreateSection("NPC / Boss ESP")
ET:CreateToggle({Name="NPC ESP (Mutantlar/Bosslar)", CurrentValue=true, Flag="ESPNpc",
    Callback=function(v) CFG.ESP.NPCs=v end})

ET:CreateSection("Kaçış Bölgesi")
ET:CreateToggle({Name="Kaçış ESP (Yeşil Daire)", CurrentValue=true, Flag="ESPEsc",
    Callback=function(v) CFG.ESP.Escape=v end})

ET:CreateSection("Görüş Alanı (Custom FOV)")
local fovTgl
fovTgl = ET:CreateToggle({Name="Custom FOV Aktif", CurrentValue=false, Flag="CFOVEn",
    Callback=function(v)
        CFG.World.CustomFOV = v
        if not v then Camera.FieldOfView = OrigFOV end
    end})
ET:CreateKeybind({Name="FOV Aç/Kapat (F4)", CurrentKeybind="F4", HoldToInteract=false, Flag="FOVKey",
    Callback=function()
        CFG.World.CustomFOV = not CFG.World.CustomFOV
        if not CFG.World.CustomFOV then Camera.FieldOfView = OrigFOV end
        if fovTgl then fovTgl:Set(CFG.World.CustomFOV) end
    end})
ET:CreateSlider({Name="FOV Değeri", Range={60,130}, Increment=1, Suffix="", CurrentValue=100, Flag="FOVVal",
    Callback=function(v) CFG.World.FOVValue=v end})

-- AİMBOT TAB
local AT = Win:CreateTab("🎯 Aimbot", 4483362458)

AT:CreateSection("Aimbot (Sağ Tık ile)")
AT:CreateToggle({Name="Aimbot Aktif", CurrentValue=false, Flag="AimEn",
    Callback=function(v) CFG.Aimbot.Enabled=v end})
AT:CreateToggle({Name="NPC'leri de Hedef Al", CurrentValue=true, Flag="AimNPC",
    Callback=function(v) CFG.Aimbot.TargetNPC=v end})
AT:CreateToggle({Name="FOV Dairesi Göster", CurrentValue=true, Flag="AimFOVShow",
    Callback=function(v) CFG.Aimbot.ShowFOV=v end})
AT:CreateSlider({Name="FOV Boyutu", Range={30,800}, Increment=5, Suffix="px", CurrentValue=150, Flag="AimFOV",
    Callback=function(v) CFG.Aimbot.FOV=v end})
AT:CreateDropdown({Name="Hedef Bölgesi", Options={"Head","HumanoidRootPart"}, CurrentOption={"Head"}, Flag="AimPart",
    Callback=function(v) CFG.Aimbot.Part=type(v)=="table" and v[1] or v end})
AT:CreateToggle({Name="Prediction", CurrentValue=false, Flag="AimPred",
    Callback=function(v) CFG.Aimbot.Predict=v end})

-- DÜNYA TAB
local WT = Win:CreateTab("🌍 Dünya", 4483345998)
WT:CreateSection("Ortam")
WT:CreateToggle({Name="Daima Gündüz", CurrentValue=false, Flag="AlwDay",
    Callback=function(v) CFG.World.AlwaysDay=v end})
WT:CreateToggle({Name="Fullbright", CurrentValue=false, Flag="Fbright",
    Callback=function(v) CFG.World.Fullbright=v end})

-- FPS / PERFORMANS TAB
local PT = Win:CreateTab("⚡ Performans", 4483345998)

PT:CreateSection("Grafik Optimizasyonu")
PT:CreateToggle({Name="Düşük Grafik (FPS Arttır)", CurrentValue=false, Flag="LowGFX",
    Callback=function(v) CFG.Perf.LowGFX=v; ApplyPerf() end})
PT:CreateToggle({Name="Gölgeleri Kapat", CurrentValue=false, Flag="NoShadows",
    Callback=function(v) CFG.Perf.NoShadows=v; ApplyPerf() end})
PT:CreateToggle({Name="Partikülleri Kaldır (Sis/Ateş)", CurrentValue=false, Flag="NoParticles",
    Callback=function(v) CFG.Perf.NoParticles=v; ApplyPerf() end})
PT:CreateToggle({Name="Sisi Kaldır", CurrentValue=false, Flag="NoFog",
    Callback=function(v) CFG.Perf.NoFog=v; ApplyPerf() end})

PT:CreateSection("FPS Limiti")
PT:CreateSlider({Name="FPS Limiti (0=Sınırsız)", Range={0,240}, Increment=10,
    Suffix="fps", CurrentValue=0, Flag="FPSCap",
    Callback=function(v) CFG.Perf.FPSCap=v; ApplyPerf() end})

Rayfield:LoadConfiguration()
ApplyPerf()  -- Kaydedilmiş performans ayarlarını yükle
print("[Mert] Project Delta V21 yüklendi! Ayarlar otomatik kaydedilir.")
