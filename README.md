-- LocalScript -> StarterPlayerScripts
-- ESP Highlight RGB automático com raio de 200 metros

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer

-- CONFIGURAÇÕES
local RADIUS_METERS = 200                 -- raio em metros (como você pediu)
local STUDS_PER_METER = 1 / 0.28          -- conversão aproximada: 1 stud ≈ 0.28m
local RADIUS_STUDS = math.floor(RADIUS_METERS * STUDS_PER_METER + 0.5)

local HIGHLIGHT_FILL_TRANSPARENCY = 0.4   -- transparência do fill (0 = opaco, 1 = invisível)
local HIGHLIGHT_OUTLINE_TRANSPARENCY = 0.0
local HIGHLIGHT_DEPTH_MODE = Enum.HighlightDepthMode.AlwaysOnTop -- ou Occluded se preferir

-- tabela para guardar highlights por personagem
local highlights = {}

-- função que cria (ou retorna) o Highlight para um Character
local function getOrCreateHighlight(character)
	if not character then return nil end
	if highlights[character] and highlights[character].Parent == character then
		return highlights[character]
	end

	-- remove highlight antigo se existir mal colocado
	if highlights[character] and highlights[character]:IsDescendantOf(game) == false then
		highlights[character]:Destroy()
		highlights[character] = nil
	end

	local hl = Instance.new("Highlight")
	hl.Name = "ESPHighlight"
	hl.Parent = character
	hl.Adornee = character -- adora o character inteiro (funciona bem)
	hl.FillTransparency = HIGHLIGHT_FILL_TRANSPARENCY
	hl.OutlineTransparency = HIGHLIGHT_OUTLINE_TRANSPARENCY
	hl.DepthMode = HIGHLIGHT_DEPTH_MODE
	hl.Enabled = false
	highlights[character] = hl
	return hl
end

-- função que pega o ponto central do personagem (HumanoidRootPart preferencial)
local function getCharacterPosition(character)
	if not character then return nil end
	local hrp = character:FindFirstChild("HumanoidRootPart")
	if hrp and hrp:IsA("BasePart") then
		return hrp.Position
	end
	-- fallback: tenta PrimaryPart
	if character.PrimaryPart and character.PrimaryPart:IsA("BasePart") then
		return character.PrimaryPart.Position
	end
	-- último recurso: tenta head
	local head = character:FindFirstChild("Head")
	if head and head:IsA("BasePart") then
		return head.Position
	end
	return nil
end

-- limpa highlights quando personagem for removido
local function onCharacterRemoving(character)
	if highlights[character] then
		highlights[character]:Destroy()
		highlights[character] = nil
	end
end

-- cria highlight para jogador quando o character aparecer
local function onPlayerAdded(player)
	local function charAdded(character)
		-- garante que highlight exista (mas ficará desativado até estar no raio)
		local hl = getOrCreateHighlight(character)
		-- quando o personagem for removido, limpamos
		character:WaitForChild("Humanoid", 5)
		character.AncestryChanged:Connect(function(_, parent)
			if not parent then
				onCharacterRemoving(character)
			end
		end)
	end

	player.CharacterAdded:Connect(charAdded)
	-- se já tiver character quando o script rodar
	if player.Character then
		charAdded(player.Character)
	end
end

-- ouvir jogadores atuais
for _, p in pairs(Players:GetPlayers()) do
	if p ~= LocalPlayer then
		onPlayerAdded(p)
	end
end
-- ouvir novos jogadores
Players.PlayerAdded:Connect(function(p)
	if p ~= LocalPlayer then
		onPlayerAdded(p)
	end
end)
-- limpar highlight de jogador que saiu
Players.PlayerRemoving:Connect(function(p)
	if p.Character then
		onCharacterRemoving(p.Character)
	end
end)

-- LOOP principal: atualiza quem está dentro do raio e atualiza cor RGB (arco-íris)
RunService.Heartbeat:Connect(function(dt)
	-- posição do local player (se disponível)
	local lpChar = LocalPlayer.Character
	if not lpChar then return end
	local lpPos = getCharacterPosition(lpChar)
	if not lpPos then return end

	-- computa cor rainbow baseada no tempo
	local hue = (tick() * 0.08) % 1        -- ajuste a velocidade mudando 0.08
	local color = Color3.fromHSV(hue, 1, 1)

	for character, hl in pairs(highlights) do
		if not character or not character.Parent then
			-- remove se caract já inválido
			if hl then hl:Destroy() end
			highlights[character] = nil
		else
			-- atualiza cor
			if hl then
				hl.FillColor = color
				hl.OutlineColor = color
			end

			-- verifica distância
			local pos = getCharacterPosition(character)
			if pos then
				local distance = (pos - lpPos).Magnitude
				if distance <= RADIUS_STUDS then
					if hl and not hl.Enabled then
						hl.Enabled = true
					end
				else
					if hl and hl.Enabled then
						hl.Enabled = false
					end
				end
			else
				-- sem posição, desativa
				if hl and hl.Enabled then
					hl.Enabled = false
				end
			end
		end
	end
end)

-- inicia automaticamente: cria highlights para futuros personagens do LocalPlayer (não mostra a si mesmo)
-- (já coberto pelas conexões acima)
