# Minecraft-AntiCheat
Minecraft Anticheat
package com.alper.anticheat;

import org.bukkit.Bukkit;
import org.bukkit.GameMode;
import org.bukkit.Material;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.block.BlockBreakEvent;
import org.bukkit.event.entity.EntityDamageByEntityEvent;
import org.bukkit.event.player.PlayerAnimationEvent;
import org.bukkit.event.player.PlayerMoveEvent;
import org.bukkit.event.player.PlayerToggleFlightEvent;
import org.bukkit.plugin.java.JavaPlugin;

import java.util.*;

public class AntiCheat extends JavaPlugin implements Listener {

    private final Map<UUID, Long> lastClickTime = new HashMap<>();
    private final Map<UUID, Integer> clickCounts = new HashMap<>();

    @Override
    public void onEnable() {
        getServer().getPluginManager().registerEvents(this, this);
        getLogger().info("AntiCheat sistemi aktif! Tüm kontroller çalışıyor.");
    }

    @Override
    public void onDisable() {
        getLogger().info("AntiCheat sistemi kapatıldı.");
    }

    // --- Uçma Kontrolü ---
    @EventHandler
    public void onPlayerToggleFlight(PlayerToggleFlightEvent event) {
        Player player = event.getPlayer();

        if (player.getGameMode() != GameMode.CREATIVE && player.getGameMode() != GameMode.SPECTATOR) {
            event.setCancelled(true);
            banPlayer(player, "Uçma hilesi tespit edildi!");
        }
    }

    // --- Hız / Fly / Elytra Kontrolü ---
    @EventHandler
    public void onPlayerMove(PlayerMoveEvent event) {
        Player player = event.getPlayer();

        if (player.getGameMode() == GameMode.SURVIVAL || player.getGameMode() == GameMode.ADVENTURE) {
            double distance = event.getFrom().distance(event.getTo());

            // Normal hız kontrolü
            if (distance > 1.2) {
                banPlayer(player, "Speed hack tespit edildi!");
            }

            // Yükseklik (fly jump) kontrolü
            if (event.getTo().getY() - event.getFrom().getY() > 1.5) {
                banPlayer(player, "Yüksek zıplama / Fly hack tespit edildi!");
            }

            // Elytra anormal hız kontrolü
            if (player.isGliding() && distance > 3.5) {
                banPlayer(player, "Elytra hız hilesi tespit edildi!");
            }

            // Su üstünde yürüme (Jesus hack)
            if (player.getLocation().getBlock().getType() == Material.WATER && player.isOnGround()) {
                banPlayer(player, "Su üstünde yürüme hilesi tespit edildi!");
            }
        }
    }

    // --- X-Ray Şüpheli Kazı Kontrolü ---
    @EventHandler
    public void onBlockBreak(BlockBreakEvent event) {
        Player player = event.getPlayer();
        Material block = event.getBlock().getType();

        if (block == Material.DIAMOND_ORE || block == Material.EMERALD_ORE || block == Material.ANCIENT_DEBRIS || block == Material.GOLD_ORE) {
            if (!player.hasPermission("anticheat.bypass")) {
                getLogger().warning(player.getName() + " şüpheli şekilde değerli maden kazdı!");
                // İstersen direkt ban:
                // banPlayer(player, "X-Ray hilesi tespit edildi!");
            }
        }
    }

    // --- KillAura / AutoClicker Kontrolü ---
    @EventHandler
    public void onPlayerClick(PlayerAnimationEvent event) {
        Player player = event.getPlayer();
        UUID uuid = player.getUniqueId();

        long currentTime = System.currentTimeMillis();
        long lastTime = lastClickTime.getOrDefault(uuid, 0L);

        if (currentTime - lastTime < 100) { // 100 ms altında tıklama (10 CPS üzeri)
            clickCounts.put(uuid, clickCounts.getOrDefault(uuid, 0) + 1);
        } else {
            clickCounts.put(uuid, 1);
        }

        lastClickTime.put(uuid, currentTime);

        if (clickCounts.get(uuid) > 15) { // 15 CPS üstü = AutoClicker şüphesi
            banPlayer(player, "AutoClicker / KillAura tespit edildi!");
        }
    }

    // --- KillAura Ekstra Kontrolü (Oyuncuya vuruş) ---
    @EventHandler
    public void onEntityDamage(EntityDamageByEntityEvent event) {
        if (event.getDamager() instanceof Player player) {
            if (event.getEntity() instanceof Player target) {
                double distance = player.getLocation().distance(target.getLocation());

                if (distance > 6.0) { // normal reach 3-4 blok, 6 üzeri şüpheli
                    banPlayer(player, "KillAura / Reach hack tespit edildi!");
                }
            }
        }
    }

    // --- Oyuncu Banlama ---
    private void banPlayer(Player player, String reason) {
        String playerName = player.getName();

        if (player.hasPermission("anticheat.bypass")) {
            return; // Adminleri etkilemez
        }

        Bukkit.getBanList(org.bukkit.BanList.Type.NAME).addBan(playerName, reason, null, "AntiCheat");
        player.kickPlayer("Banlandınız! Sebep: " + reason);
        getLogger().warning(playerName + " oyuncusu banlandı. Sebep: " + reason);
    }
}
