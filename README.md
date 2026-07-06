#region Using declarations
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Windows.Media;
using System.Xml.Serialization;
using NinjaTrader.Data;
using NinjaTrader.Gui;
using NinjaTrader.Gui.Chart;
using NinjaTrader.Gui.Tools;
using NinjaTrader.NinjaScript;
using NinjaTrader.NinjaScript.BarsTypes;
using NinjaTrader.NinjaScript.DrawingTools;
#endregion

// Declared in the global namespace (not inside NinjaTrader.NinjaScript.Indicators) so NinjaTrader's
// auto-generated MarketAnalyzerColumns/Strategies wrapper code - which lives in sibling namespaces -
// can resolve this type unqualified too.
public enum DeltaTotalVPDisplayMode
{
    Total,
    Delta,
    DeltaAndTotal
}

// How far POC/VAH/VAL rays reach to the right of their own session, once that session finalizes.
public enum LevelExtensionMode
{
    None,
    NakedExtension,
    InfinityExtend
}

namespace NinjaTrader.NinjaScript.Indicators
{
    // One Volume Profile per session, anchored over that session's own bars (like a normal session
    // volume profile) and automatically repeated for every session in the loaded chart range - not a
    // single fixed panel. "Delta and Total" mode splits each price row down the middle of the session's
    // own bar range: real Bid/Ask delta grows to the left half (green = net buying, red = net selling),
    // total volume grows to the right half - matching Deepchart's "1.1.4 Delta and Total Volume" mode.
    public class DeltaTotalVolumeProfile : Indicator
    {
        #region Fields

        private class SessionProfile
        {
            public int StartBarIndex;
            public int EndBarIndex;
            public Dictionary<double, long> Total = new Dictionary<double, long>();
            public Dictionary<double, long> Delta = new Dictionary<double, long>();
        }

        private enum LevelKind { POC, VAH, VAL, High, Low }

        private class ExtendedLevel
        {
            public double Price;
            public LevelKind Kind;
            public LevelExtensionMode Mode; // this level's own extension setting, captured when created
            public int StartBarIndex;   // bar right after the session ended
            public int FrozenBarIndex = -1; // -1 = still extending (Naked mode, untouched)
        }

        // finished sessions, oldest first, capped by MaxSessions
        private List<SessionProfile> sessions = new List<SessionProfile>();

        // the session currently being built, or null while outside the session window
        private SessionProfile currentSession;

        // POC/VAH/VAL rays extending past a finalized session, oldest first
        private List<ExtendedLevel> extendedLevels = new List<ExtendedLevel>();

        private VolumetricBarsType volumetricBars;
        private int lastCommittedVolBar = -1;
        private TimeZoneInfo selectedTimeZone = TimeZoneInfo.Local;
        private double rowSize;

        #endregion

        #region Properties

        [NinjaScriptProperty]
        [Display(Name = "Session Start Time", Description = "Start of the session used to build each day's volume profile", Order = 0, GroupName = "1. Session")]
        [PropertyEditor("NinjaTrader.Gui.Tools.TimeEditorKey")]
        public DateTime SessionStartTime { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Session End Time", Description = "End of the session - that day's profile freezes here", Order = 1, GroupName = "1. Session")]
        [PropertyEditor("NinjaTrader.Gui.Tools.TimeEditorKey")]
        public DateTime SessionEndTime { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Time Zone", Description = "Time zone the Session Start/End Time are expressed in", Order = 2, GroupName = "1. Session")]
        [TypeConverter(typeof(TimeZoneConverter))]
        public string TimeZoneSelection { get; set; }

        [Range(1, int.MaxValue)]
        [NinjaScriptProperty]
        [Display(Name = "Volumetric Bar Size (ticks)", Description = "Underlying Order Flow Volumetric bar size, in ticks, used to source real Bid/Ask volume per price", Order = 0, GroupName = "2. Volumetric Data")]
        public int VolumetricBarSize { get; set; }

        [Range(1, int.MaxValue)]
        [NinjaScriptProperty]
        [Display(Name = "Ticks Per Row", Description = "How many ticks are grouped into a single profile row", Order = 1, GroupName = "2. Volumetric Data")]
        public int TicksPerRow { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Display Mode", Description = "Total only, Delta only, or Delta (left half) + Total (right half) split rows", Order = 0, GroupName = "3. Display")]
        public DeltaTotalVPDisplayMode DisplayMode { get; set; }

        [Range(10, 300)]
        [NinjaScriptProperty]
        [Display(Name = "Max Width % of Session", Description = "How far the highest-volume row reaches across the session's own bar width. 100 = fills the session; lower values leave room so it doesn't fully cover the candles", Order = 1, GroupName = "3. Display")]
        public int ProfileScale { get; set; }

        [Range(1, 100)]
        [NinjaScriptProperty]
        [Display(Name = "Row Opacity %", Order = 2, GroupName = "3. Display")]
        public int RowOpacity { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Show POC", Description = "Highlight the row with the highest total volume", Order = 3, GroupName = "3. Display")]
        public bool ShowPOC { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Extend POC", Description = "None = row highlight only. Naked Extension = ray extends right and freezes at the first touch AFTER the session's own range. Infinity Extend = ray extends right indefinitely.", Order = 4, GroupName = "3. Display")]
        public LevelExtensionMode POCExtensionMode { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Show Session High", Description = "Highlight the session's highest traded row", Order = 5, GroupName = "3. Display")]
        public bool ShowHigh { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Extend Session High", Description = "None = row highlight only. Naked Extension = ray extends right and freezes at the first touch AFTER the session's own range. Infinity Extend = ray extends right indefinitely.", Order = 6, GroupName = "3. Display")]
        public LevelExtensionMode HighExtensionMode { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Show Session Low", Description = "Highlight the session's lowest traded row", Order = 7, GroupName = "3. Display")]
        public bool ShowLow { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Extend Session Low", Description = "None = row highlight only. Naked Extension = ray extends right and freezes at the first touch AFTER the session's own range. Infinity Extend = ray extends right indefinitely.", Order = 8, GroupName = "3. Display")]
        public LevelExtensionMode LowExtensionMode { get; set; }

        [Range(1, 99)]
        [NinjaScriptProperty]
        [Display(Name = "Value Area %", Description = "Percentage of the session's total volume contained inside the Value Area (VAH/VAL)", Order = 0, GroupName = "4. Value Area")]
        public int ValueAreaPercent { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Show VAH", Description = "Highlight the Value Area High row", Order = 1, GroupName = "4. Value Area")]
        public bool ShowVAH { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Extend VAH", Description = "None = row highlight only. Naked Extension = ray extends right and freezes at the first touch AFTER the session's own range. Infinity Extend = ray extends right indefinitely.", Order = 2, GroupName = "4. Value Area")]
        public LevelExtensionMode VAHExtensionMode { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Show VAL", Description = "Highlight the Value Area Low row", Order = 3, GroupName = "4. Value Area")]
        public bool ShowVAL { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Extend VAL", Description = "None = row highlight only. Naked Extension = ray extends right and freezes at the first touch AFTER the session's own range. Infinity Extend = ray extends right indefinitely.", Order = 4, GroupName = "4. Value Area")]
        public LevelExtensionMode VALExtensionMode { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Shade Value Area", Description = "Fill rows between VAL and VAH with a separate color instead of the regular Total Volume color", Order = 5, GroupName = "4. Value Area")]
        public bool ShadeValueArea { get; set; }

        [Range(1, 100)]
        [NinjaScriptProperty]
        [Display(Name = "Value Area Opacity %", Order = 6, GroupName = "4. Value Area")]
        public int ValueAreaOpacity { get; set; }

        [XmlIgnore]
        [Display(Name = "Delta Buy Color", Order = 0, GroupName = "5. Colors")]
        public Brush DeltaBuyBrush { get; set; }
        [Browsable(false)]
        public string DeltaBuyBrushSerializable
        {
            get { return Serialize.BrushToString(DeltaBuyBrush); }
            set { DeltaBuyBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "Delta Sell Color", Order = 1, GroupName = "5. Colors")]
        public Brush DeltaSellBrush { get; set; }
        [Browsable(false)]
        public string DeltaSellBrushSerializable
        {
            get { return Serialize.BrushToString(DeltaSellBrush); }
            set { DeltaSellBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "Total Volume Color", Order = 2, GroupName = "5. Colors")]
        public Brush TotalBrush { get; set; }
        [Browsable(false)]
        public string TotalBrushSerializable
        {
            get { return Serialize.BrushToString(TotalBrush); }
            set { TotalBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "Value Area Fill Color", Description = "Total Volume color used for rows between VAL and VAH", Order = 3, GroupName = "5. Colors")]
        public Brush ValueAreaBrush { get; set; }
        [Browsable(false)]
        public string ValueAreaBrushSerializable
        {
            get { return Serialize.BrushToString(ValueAreaBrush); }
            set { ValueAreaBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "POC Color", Order = 4, GroupName = "5. Colors")]
        public Brush POCBrush { get; set; }
        [Browsable(false)]
        public string POCBrushSerializable
        {
            get { return Serialize.BrushToString(POCBrush); }
            set { POCBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "VAH Color", Order = 5, GroupName = "5. Colors")]
        public Brush VAHBrush { get; set; }
        [Browsable(false)]
        public string VAHBrushSerializable
        {
            get { return Serialize.BrushToString(VAHBrush); }
            set { VAHBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "VAL Color", Order = 6, GroupName = "5. Colors")]
        public Brush VALBrush { get; set; }
        [Browsable(false)]
        public string VALBrushSerializable
        {
            get { return Serialize.BrushToString(VALBrush); }
            set { VALBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "Session High Color", Order = 7, GroupName = "5. Colors")]
        public Brush HighBrush { get; set; }
        [Browsable(false)]
        public string HighBrushSerializable
        {
            get { return Serialize.BrushToString(HighBrush); }
            set { HighBrush = Serialize.StringToBrush(value); }
        }

        [XmlIgnore]
        [Display(Name = "Session Low Color", Order = 8, GroupName = "5. Colors")]
        public Brush LowBrush { get; set; }
        [Browsable(false)]
        public string LowBrushSerializable
        {
            get { return Serialize.BrushToString(LowBrush); }
            set { LowBrush = Serialize.StringToBrush(value); }
        }

        [Range(1, 100)]
        [NinjaScriptProperty]
        [Display(Name = "Max Sessions Kept", Description = "Older sessions beyond this count stop being rendered/kept in memory", Order = 0, GroupName = "6. Performance")]
        public int MaxSessions { get; set; }

        #endregion

        #region Time Zone Converter

        private class TimeZoneConverter : TypeConverter
        {
            public override bool GetStandardValuesSupported(ITypeDescriptorContext context) => true;

            public override StandardValuesCollection GetStandardValues(ITypeDescriptorContext context)
            {
                return new StandardValuesCollection(new[] {
                    "Local", "EST", "CST", "MST", "PST", "UTC",
                    "GMT", "CET", "EET", "JST", "AEST"
                });
            }

            public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
            {
                return sourceType == typeof(string) || base.CanConvertFrom(context, sourceType);
            }

            public override object ConvertFrom(ITypeDescriptorContext context, System.Globalization.CultureInfo culture, object value)
            {
                if (value is string str)
                    return str;
                return base.ConvertFrom(context, culture, value);
            }
        }

        #endregion

        protected override void OnStateChange()
        {
            if (State == State.SetDefaults)
            {
                Description = "One Volume Profile per session, anchored over that session's own bars (repeated automatically for every session loaded on the chart), with real Bid/Ask Delta on the left half and Total Volume on the right half (Deepchart-style 'Delta and Total Volume' mode).";
                Name = "DeltaTotalVolumeProfile";
                Calculate = Calculate.OnEachTick;
                IsOverlay = true;
                DisplayInDataBox = false;
                PaintPriceMarkers = false;
                IsSuspendedWhileInactive = true;

                SessionStartTime = new DateTime(2000, 1, 1, 9, 30, 0);
                SessionEndTime = new DateTime(2000, 1, 1, 16, 0, 0);
                TimeZoneSelection = "Local";

                VolumetricBarSize = 500;
                TicksPerRow = 4;

                DisplayMode = DeltaTotalVPDisplayMode.DeltaAndTotal;
                ProfileScale = 100;
                RowOpacity = 85;
                ShowPOC = true;
                POCExtensionMode = LevelExtensionMode.None;
                ShowHigh = true;
                HighExtensionMode = LevelExtensionMode.None;
                ShowLow = true;
                LowExtensionMode = LevelExtensionMode.None;

                ValueAreaPercent = 70;
                ShowVAH = true;
                VAHExtensionMode = LevelExtensionMode.None;
                ShowVAL = true;
                VALExtensionMode = LevelExtensionMode.None;
                ShadeValueArea = true;
                ValueAreaOpacity = 85;

                DeltaBuyBrush = Brushes.LimeGreen;
                DeltaSellBrush = Brushes.Crimson;
                TotalBrush = Brushes.DodgerBlue;
                ValueAreaBrush = Brushes.MediumPurple;
                POCBrush = Brushes.Yellow;
                VAHBrush = Brushes.DeepSkyBlue;
                VALBrush = Brushes.DeepSkyBlue;
                HighBrush = Brushes.White;
                LowBrush = Brushes.White;

                MaxSessions = 10;
            }
            else if (State == State.Configure)
            {
                AddVolumetric(Instrument.FullName, BarsPeriodType.Tick, VolumetricBarSize, VolumetricDeltaType.BidAsk, TicksPerRow);
            }
            else if (State == State.DataLoaded)
            {
                try
                {
                    switch (TimeZoneSelection)
                    {
                        case "Local": selectedTimeZone = TimeZoneInfo.Local; break;
                        case "EST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time"); break;
                        case "CST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Central Standard Time"); break;
                        case "MST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Mountain Standard Time"); break;
                        case "PST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Pacific Standard Time"); break;
                        case "UTC": selectedTimeZone = TimeZoneInfo.Utc; break;
                        case "GMT": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("GMT Standard Time"); break;
                        case "CET": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Central European Standard Time"); break;
                        case "EET": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("E. Europe Standard Time"); break;
                        case "JST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time"); break;
                        case "AEST": selectedTimeZone = TimeZoneInfo.FindSystemTimeZoneById("AUS Eastern Standard Time"); break;
                        default: selectedTimeZone = TimeZoneInfo.Local; break;
                    }
                }
                catch (TimeZoneNotFoundException)
                {
                    selectedTimeZone = TimeZoneInfo.Local;
                    Print("DeltaTotalVolumeProfile: fuso horario selecionado nao encontrado, usando Local.");
                }

                rowSize = TickSize * Math.Max(1, TicksPerRow);
                volumetricBars = BarsArray[1].BarsType as VolumetricBarsType;

                if (volumetricBars == null)
                    Print("DeltaTotalVolumeProfile: a serie volumetrica nao pode ser carregada - o feed de dados pode nao fornecer Bid/Ask real. Requer Tick Replay para historico.");
            }
        }

        protected override void OnBarUpdate()
        {
            if (CurrentBars.Length < 2 || CurrentBars[0] < 0 || CurrentBars[1] < 0)
                return;

            if (BarsInProgress == 0)
            {
                HandleSessionBoundary(Times[0][0]);
                UpdateExtendedLevels();
            }
            else if (BarsInProgress == 1)
            {
                CommitClosedVolumetricBars();
            }
        }

        private void HandleSessionBoundary(DateTime primaryBarTime)
        {
            DateTime adjusted = TimeZoneInfo.ConvertTime(primaryBarTime, TimeZoneInfo.Local, selectedTimeZone);
            bool nowInSession = InSession(adjusted);

            if (nowInSession && currentSession == null)
            {
                currentSession = new SessionProfile { StartBarIndex = CurrentBar, EndBarIndex = CurrentBar };
                // anything already inside the currently-forming volumetric bar belongs to the new session
                lastCommittedVolBar = CurrentBars[1] - 1;
            }
            else if (!nowInSession && currentSession != null)
            {
                sessions.Add(currentSession);
                TrimOldSessions();
                AddExtendedLevelsForSession(currentSession);
                currentSession = null;
            }
            else if (nowInSession && currentSession != null)
            {
                currentSession.EndBarIndex = CurrentBar;
            }
        }

        private void AddExtendedLevelsForSession(SessionProfile session)
        {
            bool anyExtension = POCExtensionMode != LevelExtensionMode.None || VAHExtensionMode != LevelExtensionMode.None ||
                VALExtensionMode != LevelExtensionMode.None || HighExtensionMode != LevelExtensionMode.None || LowExtensionMode != LevelExtensionMode.None;
            if (!anyExtension)
                return;

            if (!TryComputePOC(session.Total, out double poc, out long maxTotal))
                return;

            // must match RenderSession's rightX bar index exactly, otherwise the ray starts a few
            // bars off from where the session's own profile box ends and leaves a visible seam
            int totalBars = Math.Max(1, session.EndBarIndex - session.StartBarIndex + 1);
            int scaledBars = Math.Max(1, (int)Math.Round(totalBars * (ProfileScale / 100.0)));
            int rayStart = session.StartBarIndex + scaledBars;

            if (ShowPOC && POCExtensionMode != LevelExtensionMode.None)
                extendedLevels.Add(new ExtendedLevel { Price = poc, Kind = LevelKind.POC, Mode = POCExtensionMode, StartBarIndex = rayStart });

            if (ShowVAH || ShowVAL)
            {
                ComputeValueArea(session.Total, poc, out double vah, out double val);
                if (ShowVAH && VAHExtensionMode != LevelExtensionMode.None)
                    extendedLevels.Add(new ExtendedLevel { Price = vah, Kind = LevelKind.VAH, Mode = VAHExtensionMode, StartBarIndex = rayStart });
                if (ShowVAL && VALExtensionMode != LevelExtensionMode.None)
                    extendedLevels.Add(new ExtendedLevel { Price = val, Kind = LevelKind.VAL, Mode = VALExtensionMode, StartBarIndex = rayStart });
            }

            if (ShowHigh && HighExtensionMode != LevelExtensionMode.None)
                extendedLevels.Add(new ExtendedLevel { Price = GetSessionHighPrice(session.Total), Kind = LevelKind.High, Mode = HighExtensionMode, StartBarIndex = rayStart });

            if (ShowLow && LowExtensionMode != LevelExtensionMode.None)
                extendedLevels.Add(new ExtendedLevel { Price = GetSessionLowPrice(session.Total), Kind = LevelKind.Low, Mode = LowExtensionMode, StartBarIndex = rayStart });

            while (extendedLevels.Count > MaxSessions * 5)
                extendedLevels.RemoveAt(0);
        }

        private void UpdateExtendedLevels()
        {
            if (extendedLevels.Count == 0)
                return;

            double hi = High[0];
            double lo = Low[0];

            foreach (ExtendedLevel lvl in extendedLevels)
            {
                if (lvl.Mode != LevelExtensionMode.NakedExtension || lvl.FrozenBarIndex >= 0)
                    continue; // Infinity mode never freezes; Naked levels already touched stay frozen

                if (lvl.Price <= hi && lvl.Price >= lo)
                    lvl.FrozenBarIndex = CurrentBar;
            }
        }

        private bool TryComputePOC(Dictionary<double, long> total, out double pocPrice, out long maxTotal)
        {
            pocPrice = 0;
            maxTotal = 0;
            foreach (var kv in total)
            {
                if (kv.Value > maxTotal)
                {
                    maxTotal = kv.Value;
                    pocPrice = kv.Key;
                }
            }
            return maxTotal > 0;
        }

        private bool InSession(DateTime t)
        {
            TimeSpan tod = t.TimeOfDay;
            TimeSpan start = SessionStartTime.TimeOfDay;
            TimeSpan end = SessionEndTime.TimeOfDay;

            if (start == end)
                return false;

            if (start < end)
                return tod >= start && tod < end;

            return tod >= start || tod < end; // sessao que cruza a meia-noite
        }

        private void TrimOldSessions()
        {
            while (sessions.Count > MaxSessions)
                sessions.RemoveAt(0);
        }

        private void CommitClosedVolumetricBars()
        {
            if (currentSession == null || volumetricBars == null)
                return;

            int closedBarIndex = CurrentBars[1] - 1; // the volumetric bar that just closed
            if (closedBarIndex <= lastCommittedVolBar)
                return;

            for (int b = lastCommittedVolBar + 1; b <= closedBarIndex; b++)
                CommitBar(b, currentSession.Total, currentSession.Delta);

            lastCommittedVolBar = closedBarIndex;
        }

        private void CommitBar(int barIndex, Dictionary<double, long> total, Dictionary<double, long> delta)
        {
            double high = BarsArray[1].GetHigh(barIndex);
            double low = BarsArray[1].GetLow(barIndex);
            AccumulateBar(barIndex, low, high, total, delta);
        }

        private void AccumulateBar(int barIndex, double low, double high,
            Dictionary<double, long> total, Dictionary<double, long> delta)
        {
            var vol = volumetricBars.Volumes[barIndex];
            if (vol == null)
                return;

            int startLevel = (int)Math.Round(low / rowSize);
            int endLevel = (int)Math.Round(high / rowSize);

            for (int level = startLevel; level <= endLevel; level++)
            {
                double price = RoundToTick(level * rowSize);

                long t = vol.GetTotalVolumeForPrice(price);
                if (t <= 0)
                    continue;

                long d = vol.GetDeltaForPrice(price);

                total[price] = total.TryGetValue(price, out long existingTotal) ? existingTotal + t : t;
                delta[price] = delta.TryGetValue(price, out long existingDelta) ? existingDelta + d : d;
            }
        }

        private double RoundToTick(double price)
        {
            return Math.Round(price / TickSize) * TickSize;
        }

        protected override void OnRender(ChartControl chartControl, ChartScale chartScale)
        {
            base.OnRender(chartControl, chartScale);

            if (volumetricBars == null)
                return;

            float opacity = RowOpacity / 100f;
            float vaOpacity = ValueAreaOpacity / 100f;

            using (SharpDX.Direct2D1.Brush deltaBuyDx = DeltaBuyBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush deltaSellDx = DeltaSellBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush totalDx = TotalBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush vaFillDx = ValueAreaBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush pocDx = POCBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush vahDx = VAHBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush valDx = VALBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush highDx = HighBrush.ToDxBrush(RenderTarget))
            using (SharpDX.Direct2D1.Brush lowDx = LowBrush.ToDxBrush(RenderTarget))
            {
                deltaBuyDx.Opacity = opacity;
                deltaSellDx.Opacity = opacity;
                totalDx.Opacity = opacity;
                vaFillDx.Opacity = vaOpacity;

                foreach (SessionProfile session in sessions)
                    RenderSession(session, false, chartControl, chartScale, deltaBuyDx, deltaSellDx, totalDx, vaFillDx, pocDx, vahDx, valDx, highDx, lowDx);

                if (currentSession != null)
                    RenderSession(currentSession, true, chartControl, chartScale, deltaBuyDx, deltaSellDx, totalDx, vaFillDx, pocDx, vahDx, valDx, highDx, lowDx);

                if (extendedLevels.Count > 0)
                    RenderExtendedLevels(chartControl, chartScale, pocDx, vahDx, valDx, highDx, lowDx);
            }
        }

        private void RenderExtendedLevels(ChartControl chartControl, ChartScale chartScale,
            SharpDX.Direct2D1.Brush pocDx, SharpDX.Direct2D1.Brush vahDx, SharpDX.Direct2D1.Brush valDx,
            SharpDX.Direct2D1.Brush highDx, SharpDX.Direct2D1.Brush lowDx)
        {
            float panelRight = ChartPanel.X + ChartPanel.W;

            foreach (ExtendedLevel lvl in extendedLevels)
            {
                try
                {
                    float y = chartScale.GetYByValue(lvl.Price);
                    if (y < ChartPanel.Y - 2 || y > ChartPanel.Y + ChartPanel.H + 2)
                        continue;

                    float startX = chartControl.GetXByBarIndex(ChartBars, lvl.StartBarIndex);

                    float endX;
                    if (lvl.Mode == LevelExtensionMode.InfinityExtend)
                        endX = panelRight;
                    else if (lvl.FrozenBarIndex >= 0)
                        endX = chartControl.GetXByBarIndex(ChartBars, lvl.FrozenBarIndex);
                    else
                        endX = chartControl.GetXByBarIndex(ChartBars, CurrentBar);

                    SharpDX.Direct2D1.Brush brush;
                    switch (lvl.Kind)
                    {
                        case LevelKind.POC: brush = pocDx; break;
                        case LevelKind.VAH: brush = vahDx; break;
                        case LevelKind.VAL: brush = valDx; break;
                        case LevelKind.High: brush = highDx; break;
                        default: brush = lowDx; break;
                    }
                    RenderTarget.DrawLine(new SharpDX.Vector2(startX, y), new SharpDX.Vector2(endX, y), brush, 2f);
                }
                catch
                {
                    // bar index momentarily outside the loaded range - skip, next frame retries
                }
            }
        }

        private void RenderSession(SessionProfile session, bool isLive, ChartControl chartControl, ChartScale chartScale,
            SharpDX.Direct2D1.Brush deltaBuyDx, SharpDX.Direct2D1.Brush deltaSellDx, SharpDX.Direct2D1.Brush totalDx, SharpDX.Direct2D1.Brush vaFillDx,
            SharpDX.Direct2D1.Brush pocDx, SharpDX.Direct2D1.Brush vahDx, SharpDX.Direct2D1.Brush valDx,
            SharpDX.Direct2D1.Brush highDx, SharpDX.Direct2D1.Brush lowDx)
        {
            try
            {
                Dictionary<double, long> total = new Dictionary<double, long>(session.Total);
                Dictionary<double, long> delta = new Dictionary<double, long>(session.Delta);

                if (isLive)
                    MergeLiveBar(total, delta);

                if (total.Count == 0)
                    return;

                if (!TryComputePOC(total, out double pocPrice, out long maxTotal))
                    return;

                long maxAbsDelta = 0;
                foreach (var kv in delta)
                {
                    long a = Math.Abs(kv.Value);
                    if (a > maxAbsDelta)
                        maxAbsDelta = a;
                }

                double vah = pocPrice, val = pocPrice;
                if (ShowVAH || ShowVAL || ShadeValueArea)
                    ComputeValueArea(total, pocPrice, out vah, out val);

                double sessionHigh = GetSessionHighPrice(total);
                double sessionLow = GetSessionLowPrice(total);

                int totalBars = Math.Max(1, session.EndBarIndex - session.StartBarIndex + 1);
                int scaledBars = Math.Max(1, (int)Math.Round(totalBars * (ProfileScale / 100.0)));
                int farLeftBarIndex = Math.Max(0, session.StartBarIndex - scaledBars);

                float leftX = chartControl.GetXByBarIndex(ChartBars, session.StartBarIndex);
                float rightX = chartControl.GetXByBarIndex(ChartBars, session.StartBarIndex + scaledBars);
                // Delta's own mirrored budget to the left of leftX, so the delta/total split (leftX)
                // lands exactly on the session's first bar - same anchor Total-only mode uses - instead
                // of drifting to the middle of the session width.
                float farLeftX = chartControl.GetXByBarIndex(ChartBars, farLeftBarIndex);

                float rowHeight = Math.Max(1f, chartScale.GetPixelsForDistance(rowSize));

                foreach (var kv in total)
                {
                    double price = kv.Key;
                    long totalVol = kv.Value;
                    delta.TryGetValue(price, out long deltaVol);

                    float y = chartScale.GetYByValue(price);
                    if (y < ChartPanel.Y - rowHeight || y > ChartPanel.Y + ChartPanel.H + rowHeight)
                        continue;

                    float top = y - rowHeight / 2f;
                    bool inValueArea = ShadeValueArea && price >= val && price <= vah;
                    SharpDX.Direct2D1.Brush rowTotalDx = inValueArea ? vaFillDx : totalDx;

                    if (DisplayMode == DeltaTotalVPDisplayMode.DeltaAndTotal)
                    {
                        float totalW = (rightX - leftX) * (float)totalVol / maxTotal;
                        SharpDX.RectangleF totalRect = new SharpDX.RectangleF(leftX, top, totalW, rowHeight);
                        RenderTarget.FillRectangle(totalRect, rowTotalDx);

                        if (maxAbsDelta > 0)
                        {
                            float deltaW = (leftX - farLeftX) * Math.Abs((float)deltaVol) / maxAbsDelta;
                            SharpDX.RectangleF deltaRect = new SharpDX.RectangleF(leftX - deltaW, top, deltaW, rowHeight);
                            RenderTarget.FillRectangle(deltaRect, deltaVol >= 0 ? deltaBuyDx : deltaSellDx);
                        }
                    }
                    else if (DisplayMode == DeltaTotalVPDisplayMode.Total)
                    {
                        float totalW = (rightX - leftX) * (float)totalVol / maxTotal;
                        SharpDX.RectangleF totalRect = new SharpDX.RectangleF(leftX, top, totalW, rowHeight);
                        RenderTarget.FillRectangle(totalRect, rowTotalDx);
                    }
                    else // Delta only
                    {
                        if (maxAbsDelta <= 0)
                            continue;
                        float deltaW = (rightX - leftX) * Math.Abs((float)deltaVol) / maxAbsDelta;
                        SharpDX.RectangleF deltaRect = new SharpDX.RectangleF(leftX, top, deltaW, rowHeight);
                        RenderTarget.FillRectangle(deltaRect, deltaVol >= 0 ? deltaBuyDx : deltaSellDx);
                    }

                    if (ShowPOC && price == pocPrice)
                    {
                        SharpDX.RectangleF pocRect = new SharpDX.RectangleF(leftX, top, rightX - leftX, rowHeight);
                        RenderTarget.DrawRectangle(pocRect, pocDx, 1.5f);
                    }

                    if (ShowVAH && price == vah)
                    {
                        SharpDX.RectangleF vahRect = new SharpDX.RectangleF(leftX, top, rightX - leftX, rowHeight);
                        RenderTarget.DrawRectangle(vahRect, vahDx, 1f);
                    }

                    if (ShowVAL && price == val)
                    {
                        SharpDX.RectangleF valRect = new SharpDX.RectangleF(leftX, top, rightX - leftX, rowHeight);
                        RenderTarget.DrawRectangle(valRect, valDx, 1f);
                    }

                    if (ShowHigh && price == sessionHigh)
                    {
                        SharpDX.RectangleF highRect = new SharpDX.RectangleF(leftX, top, rightX - leftX, rowHeight);
                        RenderTarget.DrawRectangle(highRect, highDx, 1f);
                    }

                    if (ShowLow && price == sessionLow)
                    {
                        SharpDX.RectangleF lowRect = new SharpDX.RectangleF(leftX, top, rightX - leftX, rowHeight);
                        RenderTarget.DrawRectangle(lowRect, lowDx, 1f);
                    }
                }

                if (DisplayMode == DeltaTotalVPDisplayMode.DeltaAndTotal)
                {
                    float topY = chartScale.GetYByValue(sessionHigh);
                    float botY = chartScale.GetYByValue(sessionLow);
                    using (SharpDX.Direct2D1.Brush separatorDx = new SharpDX.Direct2D1.SolidColorBrush(RenderTarget, new SharpDX.Color(128, 128, 128, 90)))
                    {
                        RenderTarget.DrawLine(new SharpDX.Vector2(leftX, topY), new SharpDX.Vector2(leftX, botY), separatorDx, 1f);
                    }
                }
            }
            catch
            {
                // bar index can momentarily fall outside the currently loaded range while the chart
                // is scrolling/loading history - skip this session's render pass, next frame retries
            }
        }

        private void ComputeValueArea(Dictionary<double, long> total, double pocPrice, out double vah, out double val)
        {
            List<double> sortedPrices = new List<double>(total.Keys);
            sortedPrices.Sort();

            int pocIndex = sortedPrices.IndexOf(pocPrice);
            if (pocIndex < 0)
            {
                vah = val = pocPrice;
                return;
            }

            long totalVolume = 0;
            foreach (var kv in total)
                totalVolume += kv.Value;

            double target = totalVolume * (ValueAreaPercent / 100.0);
            long vaVolume = total[pocPrice];
            int lowIdx = pocIndex, highIdx = pocIndex;

            while (vaVolume < target && (lowIdx > 0 || highIdx < sortedPrices.Count - 1))
            {
                long volAbove = highIdx < sortedPrices.Count - 1 ? total[sortedPrices[highIdx + 1]] : -1;
                long volBelow = lowIdx > 0 ? total[sortedPrices[lowIdx - 1]] : -1;

                if (volAbove >= volBelow)
                {
                    highIdx++;
                    vaVolume += volAbove;
                }
                else
                {
                    lowIdx--;
                    vaVolume += volBelow;
                }
            }

            vah = sortedPrices[highIdx];
            val = sortedPrices[lowIdx];
        }

        private double GetSessionHighPrice(Dictionary<double, long> total)
        {
            double max = double.MinValue;
            foreach (var kv in total)
                if (kv.Key > max) max = kv.Key;
            return max;
        }

        private double GetSessionLowPrice(Dictionary<double, long> total)
        {
            double min = double.MaxValue;
            foreach (var kv in total)
                if (kv.Key < min) min = kv.Key;
            return min;
        }

        private void MergeLiveBar(Dictionary<double, long> total, Dictionary<double, long> delta)
        {
            if (volumetricBars == null || CurrentBars.Length < 2 || CurrentBars[1] < 0)
                return;

            int liveIndex = CurrentBars[1];
            if (liveIndex <= lastCommittedVolBar)
                return;

            double high = Highs[1][0];
            double low = Lows[1][0];
            AccumulateBar(liveIndex, low, high, total, delta);
        }
    }
}

#region NinjaScript generated code. Neither change nor remove.

namespace NinjaTrader.NinjaScript.Indicators
{
	public partial class Indicator : NinjaTrader.Gui.NinjaScript.IndicatorRenderBase
	{
		private DeltaTotalVolumeProfile[] cacheDeltaTotalVolumeProfile;
		public DeltaTotalVolumeProfile DeltaTotalVolumeProfile(DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			return DeltaTotalVolumeProfile(Input, sessionStartTime, sessionEndTime, timeZoneSelection, volumetricBarSize, ticksPerRow, displayMode, profileScale, rowOpacity, showPOC, pOCExtensionMode, showHigh, highExtensionMode, showLow, lowExtensionMode, valueAreaPercent, showVAH, vAHExtensionMode, showVAL, vALExtensionMode, shadeValueArea, valueAreaOpacity, maxSessions);
		}

		public DeltaTotalVolumeProfile DeltaTotalVolumeProfile(ISeries<double> input, DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			if (cacheDeltaTotalVolumeProfile != null)
				for (int idx = 0; idx < cacheDeltaTotalVolumeProfile.Length; idx++)
					if (cacheDeltaTotalVolumeProfile[idx] != null && cacheDeltaTotalVolumeProfile[idx].SessionStartTime == sessionStartTime && cacheDeltaTotalVolumeProfile[idx].SessionEndTime == sessionEndTime && cacheDeltaTotalVolumeProfile[idx].TimeZoneSelection == timeZoneSelection && cacheDeltaTotalVolumeProfile[idx].VolumetricBarSize == volumetricBarSize && cacheDeltaTotalVolumeProfile[idx].TicksPerRow == ticksPerRow && cacheDeltaTotalVolumeProfile[idx].DisplayMode == displayMode && cacheDeltaTotalVolumeProfile[idx].ProfileScale == profileScale && cacheDeltaTotalVolumeProfile[idx].RowOpacity == rowOpacity && cacheDeltaTotalVolumeProfile[idx].ShowPOC == showPOC && cacheDeltaTotalVolumeProfile[idx].POCExtensionMode == pOCExtensionMode && cacheDeltaTotalVolumeProfile[idx].ShowHigh == showHigh && cacheDeltaTotalVolumeProfile[idx].HighExtensionMode == highExtensionMode && cacheDeltaTotalVolumeProfile[idx].ShowLow == showLow && cacheDeltaTotalVolumeProfile[idx].LowExtensionMode == lowExtensionMode && cacheDeltaTotalVolumeProfile[idx].ValueAreaPercent == valueAreaPercent && cacheDeltaTotalVolumeProfile[idx].ShowVAH == showVAH && cacheDeltaTotalVolumeProfile[idx].VAHExtensionMode == vAHExtensionMode && cacheDeltaTotalVolumeProfile[idx].ShowVAL == showVAL && cacheDeltaTotalVolumeProfile[idx].VALExtensionMode == vALExtensionMode && cacheDeltaTotalVolumeProfile[idx].ShadeValueArea == shadeValueArea && cacheDeltaTotalVolumeProfile[idx].ValueAreaOpacity == valueAreaOpacity && cacheDeltaTotalVolumeProfile[idx].MaxSessions == maxSessions && cacheDeltaTotalVolumeProfile[idx].EqualsInput(input))
						return cacheDeltaTotalVolumeProfile[idx];
			return CacheIndicator<DeltaTotalVolumeProfile>(new DeltaTotalVolumeProfile(){ SessionStartTime = sessionStartTime, SessionEndTime = sessionEndTime, TimeZoneSelection = timeZoneSelection, VolumetricBarSize = volumetricBarSize, TicksPerRow = ticksPerRow, DisplayMode = displayMode, ProfileScale = profileScale, RowOpacity = rowOpacity, ShowPOC = showPOC, POCExtensionMode = pOCExtensionMode, ShowHigh = showHigh, HighExtensionMode = highExtensionMode, ShowLow = showLow, LowExtensionMode = lowExtensionMode, ValueAreaPercent = valueAreaPercent, ShowVAH = showVAH, VAHExtensionMode = vAHExtensionMode, ShowVAL = showVAL, VALExtensionMode = vALExtensionMode, ShadeValueArea = shadeValueArea, ValueAreaOpacity = valueAreaOpacity, MaxSessions = maxSessions }, input, ref cacheDeltaTotalVolumeProfile);
		}
	}
}

namespace NinjaTrader.NinjaScript.MarketAnalyzerColumns
{
	public partial class MarketAnalyzerColumn : MarketAnalyzerColumnBase
	{
		public Indicators.DeltaTotalVolumeProfile DeltaTotalVolumeProfile(DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			return indicator.DeltaTotalVolumeProfile(Input, sessionStartTime, sessionEndTime, timeZoneSelection, volumetricBarSize, ticksPerRow, displayMode, profileScale, rowOpacity, showPOC, pOCExtensionMode, showHigh, highExtensionMode, showLow, lowExtensionMode, valueAreaPercent, showVAH, vAHExtensionMode, showVAL, vALExtensionMode, shadeValueArea, valueAreaOpacity, maxSessions);
		}

		public Indicators.DeltaTotalVolumeProfile DeltaTotalVolumeProfile(ISeries<double> input , DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			return indicator.DeltaTotalVolumeProfile(input, sessionStartTime, sessionEndTime, timeZoneSelection, volumetricBarSize, ticksPerRow, displayMode, profileScale, rowOpacity, showPOC, pOCExtensionMode, showHigh, highExtensionMode, showLow, lowExtensionMode, valueAreaPercent, showVAH, vAHExtensionMode, showVAL, vALExtensionMode, shadeValueArea, valueAreaOpacity, maxSessions);
		}
	}
}

namespace NinjaTrader.NinjaScript.Strategies
{
	public partial class Strategy : NinjaTrader.Gui.NinjaScript.StrategyRenderBase
	{
		public Indicators.DeltaTotalVolumeProfile DeltaTotalVolumeProfile(DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			return indicator.DeltaTotalVolumeProfile(Input, sessionStartTime, sessionEndTime, timeZoneSelection, volumetricBarSize, ticksPerRow, displayMode, profileScale, rowOpacity, showPOC, pOCExtensionMode, showHigh, highExtensionMode, showLow, lowExtensionMode, valueAreaPercent, showVAH, vAHExtensionMode, showVAL, vALExtensionMode, shadeValueArea, valueAreaOpacity, maxSessions);
		}

		public Indicators.DeltaTotalVolumeProfile DeltaTotalVolumeProfile(ISeries<double> input , DateTime sessionStartTime, DateTime sessionEndTime, string timeZoneSelection, int volumetricBarSize, int ticksPerRow, DeltaTotalVPDisplayMode displayMode, int profileScale, int rowOpacity, bool showPOC, LevelExtensionMode pOCExtensionMode, bool showHigh, LevelExtensionMode highExtensionMode, bool showLow, LevelExtensionMode lowExtensionMode, int valueAreaPercent, bool showVAH, LevelExtensionMode vAHExtensionMode, bool showVAL, LevelExtensionMode vALExtensionMode, bool shadeValueArea, int valueAreaOpacity, int maxSessions)
		{
			return indicator.DeltaTotalVolumeProfile(input, sessionStartTime, sessionEndTime, timeZoneSelection, volumetricBarSize, ticksPerRow, displayMode, profileScale, rowOpacity, showPOC, pOCExtensionMode, showHigh, highExtensionMode, showLow, lowExtensionMode, valueAreaPercent, showVAH, vAHExtensionMode, showVAL, vALExtensionMode, shadeValueArea, valueAreaOpacity, maxSessions);
		}
	}
}

#endregion
