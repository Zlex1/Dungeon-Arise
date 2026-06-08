-- Auto Dungeon Hub v19 - Teleporta Imediato, Depois Espera
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- ============================================
-- CONFIGURAÇÕES
-- ============================================
local ConfigFileName = "DungeonHub_Config.json"
local HttpService = game:GetService("HttpService")

local Config = {
    AutoStart = false,
    Difficulty = "Easy",
    WaitTime = 120  -- Tempo de espera DEPOIS do teleporte
}

local function SaveConfig()
    if writefile then
        pcall(function() writefile(ConfigFileName, HttpService:JSONEncode(Config)) end)
    end
end

local function LoadConfig()
    if isfile and isfile(ConfigFileName) then
        pcall(function()
            local data = HttpService:JSONDecode(readfile(ConfigFileName))
            for k,v in pairs(data) do Config[k] = v end
        end)
    end
end
LoadConfig()

-- ============================================
-- INTERFACE
-- ============================================
local Window = Rayfield:CreateWindow({
    Name = "Auto Dungeon Hub",
    LoadingTitle = "Carregando...",
    ConfigurationSaving = {Enabled = false}
})

local MainTab = Window:CreateTab("Principal", 4483362458)

local StatusLabel = MainTab:CreateLabel("Status: Desativado")
local TimerLabel = MainTab:CreateLabel("Timer: --")

MainTab:CreateToggle({
    Name = "Ativar Auto Dungeon",
    CurrentValue = Config.AutoStart,
    Callback = function(v)
        Config.AutoStart = v
        SaveConfig()
        print("[DEBUG] AutoStart: " .. tostring(v))
    end
})

MainTab:CreateDropdown({
    Name = "Dificuldade",
    Options = {"Easy", "Normal", "Hard", "Insane" },
    CurrentOption = {Config.Difficulty},
    Callback = function(v)
        Config.Difficulty = v[1]
        SaveConfig()
    end
})

MainTab:CreateSlider({
    Name = "Tempo de espera (segundos)",
    Range = {30, 300},
    Increment = 10,
    Suffix = "s",
    CurrentValue = Config.WaitTime,
    Callback = function(v)
        Config.WaitTime = v
        SaveConfig()
    end
})

-- ============================================
-- FUNÇÕES
-- ============================================

-- Teleportar para o Boss (IMEDIATO)
function TeleportToBoss()
    local player = game.Players.LocalPlayer
    local char = player.Character
    if not char then
        player.CharacterAdded:Wait()
        char = player.Character
        task.wait(1)
    end
    
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if hrp then
        local bossPosition = CFrame.new(215.72, 1594.49, -4294.38)
        hrp.CFrame = bossPosition
        print("[📍] Teleportado para o Boss IMEDIATAMENTE!")
        return true
    end
    return false
end

-- Enviar CREATE
function SendCreate()
    pcall(function()
        local bridge = game:GetService("ReplicatedStorage"):FindFirstChild("BridgeNet2")
        if bridge then
            local remote = bridge:FindFirstChild("dataRemoteEvent")
            if remote then
                local args = {
                    [1] = {
                        [1] = {
                            ["Event"] = "BossRushAction",
                            ["Action"] = "Create"
                        },
                        [2] = "\17"
                    }
                }
                remote:FireServer(unpack(args))
                print("[✓] CREATE enviado!")
            end
        end
    end)
end

-- Enviar START
function SendStart()
    pcall(function()
        local bridge = game:GetService("ReplicatedStorage"):FindFirstChild("BridgeNet2")
        if bridge then
            local remote = bridge:FindFirstChild("dataRemoteEvent")
            if remote then
                local args = {
                    [1] = {
                        [1] = {
                            ["Dungeon"] = 2394154405,
                            ["Action"] = "Start",
                            ["Diff"] = Config.Difficulty,
                            ["Event"] = "BossRushAction"
                        },
                        [2] = "\17"
                    }
                }
                remote:FireServer(unpack(args))
                print("[✓] START enviado! Dificuldade: " .. Config.Difficulty)
            end
        end
    end)
end

-- Executar CREATE + START (depois da espera)
function ExecuteCreateAndStart()
    print("[▶] ===== EXECUTANDO CREATE + START =====")
    StatusLabel:Set("Status: Enviando CREATE...")
    SendCreate()
    task.wait(1)
    
    StatusLabel:Set("Status: Enviando START...")
    SendStart()
    
    Rayfield:Notify({
        Title = "✅ Dungeon Iniciada!",
        Content = Config.Difficulty,
        Duration = 2
    })
    
    print("[▶] ===== DUNGEON INICIADA ===== ")
    StatusLabel:Set("Status: Dungeon em andamento...")
end

-- ============================================
-- LOOP PRINCIPAL
-- ============================================

local loopActive = false

task.spawn(function()
    print("[🎮] Script carregado!")
    
    while true do
        if Config.AutoStart then
            if not loopActive then
                loopActive = true
                print("[🔁] Loop ativado!")
                
                while Config.AutoStart do
                    -- PASSO 1: TELEPORTAR IMEDIATAMENTE
                    print("[📍] Teleportando para o Boss AGORA MESMO...")
                    StatusLabel:Set("Status: Teleportando para o Boss...")
                    TeleportToBoss()
                    
                    Rayfield:Notify({
                        Title = "📍 Teleportado!",
                        Content = "Aguardando " .. Config.WaitTime .. " segundos...",
                        Duration = 3
                    })
                    
                    -- PASSO 2: ESPERAR O TEMPO
                    print("[⏳] Aguardando " .. Config.WaitTime .. " segundos para enviar CREATE + START...")
                    StatusLabel:Set("Status: Aguardando " .. Config.WaitTime .. "s para iniciar dungeon")
                    
                    local startTime = tick()
                    local elapsed = 0
                    
                    while elapsed < Config.WaitTime and Config.AutoStart do
                        local remaining = Config.WaitTime - elapsed
                        local minutos = math.floor(remaining / 60)
                        local segundos = remaining % 60
                        TimerLabel:Set(string.format("Timer: %02d:%02d", minutos, segundos))
                        
                        if math.floor(remaining) % 30 == 0 and remaining > 0 then
                            print("[⏳] Enviando comandos em " .. math.floor(remaining) .. " segundos")
                            StatusLabel:Set(string.format("Status: Enviando em %ds", math.floor(remaining)))
                        end
                        
                        task.wait(1)
                        elapsed = tick() - startTime
                    end
                    
                    -- PASSO 3: SÓ DEPOIS DE ESPERAR, ENVIAR CREATE + START
                    if Config.AutoStart then
                        print("[▶] Tempo esgotado! Enviando CREATE + START...")
                        ExecuteCreateAndStart()
                        
                        -- PASSO 4: Pequena pausa antes de recomeçar o ciclo
                        print("[⏳] Aguardando 5 segundos antes de teleportar novamente...")
                        StatusLabel:Set("Status: Aguardando 5s para recomeçar")
                        
                        for i = 5, 1, -1 do
                            if not Config.AutoStart then break end
                            StatusLabel:Set(string.format("Status: Recomeçando em %ds", i))
                            task.wait(1)
                        end
                    end
                end
            end
        else
            if loopActive then
                loopActive = false
                TimerLabel:Set("Timer: --")
                StatusLabel:Set("Status: Desativado")
                print("[⏸] Loop desativado!")
            end
            task.wait(1)
        end
        task.wait(0.1)
    end
end)

-- Notificação inicial
Rayfield:Notify({
    Title = "✅ Script Carregado!",
    Content = "Ao ativar: Teleporta IMEDIATO → Espera " .. Config.WaitTime .. "s → Inicia dungeon",
    Duration = 5
})

print("[✅] Script v19 carregado!")
print("[✅] Fluxo: Teleporta AGORA → Espera " .. Config.WaitTime .. "s → CREATE + START → Espera 5s → Repete")
