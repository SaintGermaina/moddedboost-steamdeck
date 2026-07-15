--- bs-pristine/global.json	2026-07-15 22:09:21.530219489 +0900
+++ bs/global.json	2026-07-15 22:04:29.188201288 +0900
@@ -1,6 +1,6 @@
 {
   "sdk": {
-    "version": "9.0.100-preview.5.24307.3",
+    "version": "9.0.100",
     "rollForward": "latestFeature"
   }
 }
\ No newline at end of file
--- bs-pristine/src/Core/Infrastructure/Formats/PsarcFormat/PsarcPacker.cs	2026-07-15 22:09:21.104219463 +0900
+++ bs/src/Core/Infrastructure/Formats/PsarcFormat/PsarcPacker.cs	2026-07-15 22:02:07.713192480 +0900
@@ -72,7 +72,7 @@
 
         var isLinux = OperatingSystem.IsLinux();
         var tempPsarcExePath = await InitializeExecutableAsync(workingDirectory, cancellationToken);
-        var wine = isLinux ? "/usr/bin/wine" : tempPsarcExePath;
+        var wine = isLinux ? ResolveWinePath() : tempPsarcExePath;
         var wineArg = isLinux ? $"\"{tempPsarcExePath}\"" : string.Empty;
         var unpackDirectory = isLinux 
             ? Path.GetRandomFileName()
@@ -200,7 +200,7 @@
         logger.LogInformation("Constructed psarc xml metadata: {@tempString}", tempString);
         
         var isLinux = OperatingSystem.IsLinux();
-        var wine = isLinux ? "/usr/bin/wine" : tempPsarcExePath;
+        var wine = isLinux ? ResolveWinePath() : tempPsarcExePath;
         var wineArg = isLinux ? $"\"{tempPsarcExePath}\"" : string.Empty;
         
         var arguments = $"{wineArg} create --xml \"{tempMetadataPath}\"";
@@ -254,4 +254,71 @@
 
         return workingPath;
     }
+
+    /// <summary>
+    /// Locates a usable wine binary on Linux. Checks (in order): the MODDEDBOOST_WINE
+    /// environment variable, standard system paths / PATH, then Steam Proton installations
+    /// (so this works out of the box on Steam Deck / SteamOS where wine is not installed system-wide).
+    /// </summary>
+    private static string ResolveWinePath()
+    {
+        var overridePath = Environment.GetEnvironmentVariable("MODDEDBOOST_WINE");
+        if (!string.IsNullOrEmpty(overridePath) && File.Exists(overridePath))
+            return overridePath;
+
+        var candidates = new List<string> { "/usr/bin/wine", "/usr/local/bin/wine" };
+        var pathVariable = Environment.GetEnvironmentVariable("PATH") ?? string.Empty;
+        candidates.AddRange(pathVariable
+            .Split(':', StringSplitOptions.RemoveEmptyEntries)
+            .Select(directory => Path.Combine(directory, "wine")));
+
+        foreach (var candidate in candidates.Where(File.Exists))
+            return candidate;
+
+        // Steam Proton installations (Steam Deck)
+        var home = Environment.GetFolderPath(Environment.SpecialFolder.UserProfile);
+        var steamCommonDirectories = new[]
+        {
+            Path.Combine(home, ".local/share/Steam/steamapps/common"),
+            Path.Combine(home, ".steam/steam/steamapps/common"),
+        };
+
+        foreach (var commonDirectory in steamCommonDirectories)
+        {
+            if (!Directory.Exists(commonDirectory))
+                continue;
+
+            var protonWineCandidates = Directory
+                .GetDirectories(commonDirectory)
+                .Where(directory => Path.GetFileName(directory).Contains("Proton", StringComparison.OrdinalIgnoreCase))
+                .OrderByDescending(directory => directory)
+                .SelectMany(directory => new[]
+                {
+                    Path.Combine(directory, "files", "bin", "wine"),
+                    Path.Combine(directory, "dist", "bin", "wine"),
+                });
+
+            foreach (var wineCandidate in protonWineCandidates.Where(File.Exists))
+            {
+                // Proton's wine needs its bundled libraries and a writable, isolated prefix
+                var protonRoot = Path.GetFullPath(Path.Combine(Path.GetDirectoryName(wineCandidate)!, ".."));
+                var existingLdPath = Environment.GetEnvironmentVariable("LD_LIBRARY_PATH");
+                var protonLibraryPaths = $"{Path.Combine(protonRoot, "lib")}:{Path.Combine(protonRoot, "lib64")}";
+                Environment.SetEnvironmentVariable("LD_LIBRARY_PATH",
+                    string.IsNullOrEmpty(existingLdPath) ? protonLibraryPaths : $"{protonLibraryPaths}:{existingLdPath}");
+
+                if (string.IsNullOrEmpty(Environment.GetEnvironmentVariable("WINEPREFIX")))
+                {
+                    Environment.SetEnvironmentVariable("WINEPREFIX",
+                        Path.Combine(home, ".moddedboost", "wineprefix"));
+                }
+
+                return wineCandidate;
+            }
+        }
+
+        // Last resort: hope wine appears on PATH at spawn time
+        return "wine";
+    }
+
 }
