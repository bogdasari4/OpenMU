# Developing external plugins without modifying the core

OpenMU plugins can be shipped as separate assemblies so you don't have to touch the core projects or generated packet files. The server discovers external plugins through its plugin configuration and loads the specified assemblies from the `plugins/` directory at runtime.

## Prerequisites
- Create a .NET class library project which references the OpenMU packages you need (for packet handlers and views you typically reference `MUnique.OpenMU.GameServer`, `MUnique.OpenMU.GameLogic`, and `MUnique.OpenMU.Network`).
- Enable nullable reference types and match the C# language version used in the repository (`Directory.Build.props`).

## Writing the plugin
1. Define your class and mark it with the standard attributes:
   ```csharp
   using System.Runtime.InteropServices;
   using MUnique.OpenMU.GameLogic;
   using MUnique.OpenMU.GameServer.MessageHandler;
   using MUnique.OpenMU.PlugIns;

   [Guid("7E6BAA65-3B62-4C8C-9A5F-1E8F5C662D90")]
   [PlugIn("Item post request handler", "Handles C1 F3 40 item post requests without touching core assemblies.")]
   [MinimumClient(Season = 6, Episode = 3, Language = ClientLanguage.Invariant)]
   public class ExternalItemPostRequestHandlerPlugIn : IPacketHandlerPlugIn
   {
       public bool IsEncryptionExpected => false;
       public byte Key => 0xF3;
       public byte SubKey => 0x40;

       public void HandlePacket(Player player, Span<byte> packet)
       {
           // Forward to your own item post action implementation.
       }
   }
   ```
2. Implement the corresponding game logic (e.g., an action class or view plugin) in the same assembly. You can re-use existing interfaces such as `IItemPostInfoViewPlugIn` to render tooltips or `ItemPostAction` patterns to keep changes isolated.
3. Avoid modifying generated packet structs. Instead, consume the existing packet references in `MUnique.OpenMU.Network.Packets` (e.g., `ItemPostInfoRequestRef`) which exposes `SpanReader`/`SpanWriter` helpers for serialization.

## Building and deploying
1. Build your class library so it produces `<YourPlugin>.dll`.
2. Copy the built DLL into the `plugins/` folder next to the server executable (created automatically after running the server once).
3. Open the Admin Panel and create or edit the plugin configuration entry that targets your plugin interface. Set **ExternalAssemblyName** to your DLL name (for example `ExternalItemPost.dll`) and activate the plugin.
4. Restart the game server or reload plugin configuration so the PlugInManager picks up the external assembly and registers the plugin.

## Benefits of the external approach
- Keeps repository forks small because you don't need to edit core or generated files.
- Plugin assemblies can be swapped independently and toggled per game server through configuration.
- Multiple client versions can be supported by providing multiple plugin implementations annotated with the appropriate `MinimumClient` values.
