--[[
AVISO IMPORTANTE
Este script é para TESTES em um jogo que VOCÊ POSSUI ou tem AUTORIZAÇÃO.
Funciona de forma LEGÍTIMA (server-side), sem exploits.
Requisitos:
- Place seu
- Mapa chamado "Chaos"
- Tool chamada "Revolver" em ServerStorage
- Apenas admins autorizados podem usar
]]

-----------------------------
-- CONFIGURAÇÕES
-----------------------------
local MAP_NAME = "Chaos"
local TOOL_NAME = "Revolver"
local DAMAGE = 35
local FIRE_INTERVAL = 0.25

-- Defina quem pode usar (UserIds)
local ADMINS = {
    [123456789] = true, -- troque pelo seu UserId
}

-----------------------------
-- SERVER SCRIPT
-- Coloque em ServerScriptService
-----------------------------
local Players = game:GetService("Players")
local ServerStorage = game:GetService("ServerStorage")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

-- Remote
local remote = Instance.new("RemoteEvent")
remote.Name = "LoopKillRemote"
remote.Parent = ReplicatedStorage

local activeLoops = {} -- [adminPlayer] = { [targetPlayer] = true }

local function isInChaos(player)
    return workspace:FindFirstChild(MAP_NAME) ~= nil
end

local function giveRevolver(character)
    if not character then return end
    local tool = ServerStorage:FindFirstChild(TOOL_NAME)
    if tool and not character:FindFirstChild(TOOL_NAME) then
        tool:Clone().Parent = character
    end
end

local function shootAt(shooter, target)
    if not shooter.Character or not target.Character then return end
    local sHRP = shooter.Character:FindFirstChild("HumanoidRootPart")
    local tHRP = target.Character:FindFirstChild("HumanoidRootPart")
    local humanoid = target.Character:FindFirstChildOfClass("Humanoid")
    if not sHRP or not tHRP or not humanoid then return end

    local direction = (tHRP.Position - sHRP.Position)
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {shooter.Character}
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist

    local result = workspace:Raycast(sHRP.Position, direction, rayParams)
    if result and result.Instance:IsDescendantOf(target.Character) then
        humanoid:TakeDamage(DAMAGE)
    end
end

remote.OnServerEvent:Connect(function(admin, action, targets)
    if not ADMINS[admin.UserId] then return end
    if not isInChaos(admin) then return end

    activeLoops[admin] = activeLoops[admin] or {}

    if action == "START" then
        for _, target in ipairs(targets) do
            if target ~= admin then
                activeLoops[admin][target] = true
            end
        end
    elseif action == "STOP" then
        for _, target in ipairs(targets) do
            activeLoops[admin][target] = nil
        end
    end
end)

RunService.Heartbeat:Connect(function()
    for admin, targets in pairs(activeLoops) do
        if admin.Character then
            giveRevolver(admin.Character)
            for target, enabled in pairs(targets) do
                if enabled then
                    shootAt(admin, target)
                end
            end
        end
    end
end)

-----------------------------
-- LOCAL SCRIPT (GUI)
-- Coloque em StarterPlayerScripts
-----------------------------
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer
local remote = ReplicatedStorage:WaitForChild("LoopKillRemote")

-- GUI simples
local gui = Instance.new("ScreenGui", player.PlayerGui)
local frame = Instance.new("Frame", gui)
frame.Size = UDim2.fromScale(0.25, 0.5)
frame.Position = UDim2.fromScale(0.02, 0.25)
frame.BackgroundColor3 = Color3.fromRGB(25,25,25)

local layout = Instance.new("UIListLayout", frame)

local selected = {}

local function togglePlayer(p)
    selected[p] = not selected[p]
end

for _, p in ipairs(Players:GetPlayers()) do
    if p ~= player then
        local btn = Instance.new("TextButton", frame)
        btn.Text = p.Name
        btn.Size = UDim2.fromScale(1, 0.08)
        btn.MouseButton1Click:Connect(function()
            togglePlayer(p)
            btn.BackgroundColor3 = selected[p] and Color3.fromRGB(0,120,0) or Color3.fromRGB(60,60,60)
        end)
    end
end

local startBtn = Instance.new("TextButton", frame)
startBtn.Text = "INICIAR LOOP"
startBtn.Size = UDim2.fromScale(1, 0.1)
startBtn.BackgroundColor3 = Color3.fromRGB(0,150,0)

local stopBtn = Instance.new("TextButton", frame)
stopBtn.Text = "PARAR LOOP"
stopBtn.Size = UDim2.fromScale(1, 0.1)
stopBtn.BackgroundColor3 = Color3.fromRGB(150,0,0)

startBtn.MouseButton1Click:Connect(function()
    local targets = {}
    for p, v in pairs(selected) do
        if v then table.insert(targets, p) end
    end
    remote:FireServer("START", targets)
end)

stopBtn.MouseButton1Click:Connect(function()
    local targets = {}
    for p, _ in pairs(selected) do
        table.insert(targets, p)
    end
    remote:FireServer("STOP", targets)
end)
