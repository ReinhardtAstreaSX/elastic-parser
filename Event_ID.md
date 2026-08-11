PUT _ingest/pipeline/logs-system.security@custom
{
  "description": "Exclusive allowlist for critical Windows Security Event IDs. Drops everything else.",
  "processors": [
    {
      "drop": {
        "description": "Drop events NOT in the important IDs list",
        "if": """
          if (ctx?.winlog?.event_id == null) {
            return false; // Don't drop malformed logs without an ID, let them through for troubleshooting
          }
          
          def eventId = Integer.parseInt(ctx.winlog.event_id.toString());
          
          def keepList = new HashSet([
            41, 1074, 1102, 
            4624, 4625, 4648, 4657, 4672, 4688, 4697, 4698, 
            4719, 4720, 4721, 4722, 4723, 4724, 4725, 4726, 4727, 4728, 4729, 4730, 4731, 4732, 4733, 4734, 4735, 4736, 4737, 4738, 4739, 4740, 4741, 4742, 4743, 
            4754, 4755, 4756, 4757, 4758, 
            4764, 4765, 4766, 4767, 4769, 
            4781, 4782, 4783, 4784, 4785, 4786, 4787, 4788, 4789, 
            4907, 5136, 6008, 6416, 7036, 7040, 7045
          ]);
          
          // Return TRUE to drop the event if the ID is NOT in the keepList
          return !keepList.contains(eventId);
        """
      }
    }
  ]
}