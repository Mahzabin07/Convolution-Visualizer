classdef ConvolutionVisualizer < matlab.apps.AppBase
    % Properties that correspond to app components
    properties (Access = public)
        UIFigure               matlab.ui.Figure
        TabGroup               matlab.ui.container.TabGroup
        DiscreteTimeTab        matlab.ui.container.Tab
        ContinuousTimeTab      matlab.ui.container.Tab
        UIAxes1                matlab.ui.control.UIAxes
        UIAxes2                matlab.ui.control.UIAxes
        UIAxes3                matlab.ui.control.UIAxes
        UIAxes4                matlab.ui.control.UIAxes  % New axes for correlation
        % Discrete-time components
        InputVectorEditLabel   matlab.ui.control.Label
        InputVectorEdit        matlab.ui.control.EditField
        InputIndexEditLabel    matlab.ui.control.Label
        InputIndexEdit         matlab.ui.control.EditField
        ImpulseVectorEditLabel matlab.ui.control.Label
        ImpulseVectorEdit      matlab.ui.control.EditField
        ImpulseIndexEditLabel  matlab.ui.control.Label
        ImpulseIndexEdit       matlab.ui.control.EditField
        % Continuous-time components
        InputSignalDropDownLabel matlab.ui.control.Label
        InputSignalDropDown      matlab.ui.control.DropDown
        InputStartEditLabel    matlab.ui.control.Label
        InputStartEdit         matlab.ui.control.NumericEditField
        InputEndEditLabel      matlab.ui.control.Label
        InputEndEdit           matlab.ui.control.NumericEditField
        InputAmpEditLabel      matlab.ui.control.Label
        InputAmpEdit           matlab.ui.control.NumericEditField
        ImpulseSignalDropDownLabel matlab.ui.control.Label
        ImpulseSignalDropDown    matlab.ui.control.DropDown
        ImpulseStartEditLabel  matlab.ui.control.Label
        ImpulseStartEdit       matlab.ui.control.NumericEditField
        ImpulseEndEditLabel    matlab.ui.control.Label
        ImpulseEndEdit         matlab.ui.control.NumericEditField
        ImpulseAmpEditLabel    matlab.ui.control.Label
        ImpulseAmpEdit         matlab.ui.control.NumericEditField
        % Common components
        ComputeButton         matlab.ui.control.Button
        ComputeCorrButton     matlab.ui.control.Button  % New correlation button
        ResetButton           matlab.ui.control.Button
        SpeedSliderLabel      matlab.ui.control.Label
        SpeedSlider           matlab.ui.control.Slider
        AnimationCheckBox     matlab.ui.control.CheckBox
    end
    
    properties (Access = private)
        animationTimer timer
        animationStep = 1
        totalSteps = 100
        isAnimating = false
        inputSignal
        impulseResponse
        convolutionResult
        correlationResult
        inputTime
        impulseTime
        outputTime
        correlationTime
        timeStep
        showCorrelation = false
    end
    
    methods (Access = private)
        
        function computeDiscreteConvolution(app)
    % Parse input signal
    inputVec = str2num(app.InputVectorEdit.Value);
    inputIdx = str2double(app.InputIndexEdit.Value);
    
    % Parse impulse response
    impulseVec = str2num(app.ImpulseVectorEdit.Value);
    impulseIdx = str2double(app.ImpulseIndexEdit.Value);
    
    % Compute convolution
    app.convolutionResult = conv(inputVec, impulseVec);
    
    % Compute correlation (new)
    [app.correlationResult, lags] = xcorr(inputVec, impulseVec, 'none');
    app.correlationTime = lags + inputIdx + impulseIdx;
    
    % Create time axes
    inputLen = length(inputVec);
    impulseLen = length(impulseVec);
    outputLen = inputLen + impulseLen - 1;
    
    % Input signal time axis
    app.inputTime = inputIdx:inputIdx+inputLen-1;
    app.inputSignal = inputVec;
    
    % Impulse response time axis
    app.impulseTime = impulseIdx:impulseIdx+impulseLen-1;
    app.impulseResponse = impulseVec;
    
    % Output time axis
    outputStart = inputIdx + impulseIdx;
    app.outputTime = outputStart:outputStart+outputLen-1;
    
    % For animation
    app.totalSteps = outputLen + impulseLen;
    app.timeStep = 1;
    
    % Plot initial signals
    plotInitialSignals(app);
end

        
        function computeContinuousConvolution(app)
    % Parameters
    fs = 100; % Sampling frequency (Hz)
    t_start = min(app.InputStartEdit.Value, app.ImpulseStartEdit.Value);
    t_end = max(app.InputEndEdit.Value, app.ImpulseEndEdit.Value);
    
    % Create time axis for the input signal
    app.inputTime = linspace(t_start, t_end, round((t_end - t_start) * fs));
    
    % Generate input signal
    inputType = app.InputSignalDropDown.Value;
    amp = app.InputAmpEdit.Value;
    start = app.InputStartEdit.Value;
    end_ = app.InputEndEdit.Value;
    app.inputSignal = app.generateSignal(inputType, amp, start, end_, app.inputTime);
    
    % Generate impulse response
    impulseType = app.ImpulseSignalDropDown.Value;
    impulse_amp = app.ImpulseAmpEdit.Value;
    impulse_start = app.ImpulseStartEdit.Value;
    impulse_end = app.ImpulseEndEdit.Value;
    impulse_time = linspace(impulse_start, impulse_end, round((impulse_end - impulse_start) * fs));
    app.impulseResponse = app.generateSignal(impulseType, impulse_amp, impulse_start, impulse_end, impulse_time);
    app.impulseTime = impulse_time;
    
    % Compute convolution
    dt = 1/fs;
    app.convolutionResult = conv(app.inputSignal, app.impulseResponse, 'full') * dt;
    
    % Compute correlation (new)
    [app.correlationResult, lags] = xcorr(app.inputSignal, app.impulseResponse, 'none');
    app.correlationTime = lags/fs + t_start + impulse_start;
    
    % Create output time axis
    outputLen = length(app.convolutionResult);
    total_time = (t_end - t_start) + (impulse_end - impulse_start);
    app.outputTime = linspace(t_start + impulse_start, t_start + impulse_start + total_time, outputLen);
    
    % For animation
    app.totalSteps = length(app.inputTime) + length(app.impulseResponse);
    app.timeStep = (app.inputTime(2) - app.inputTime(1));
    
    % Plot initial signals
    plotInitialSignals(app);
end

        
       function signal = generateSignal(~, signalType, amplitude, startTime, endTime, timeVector)
    signal = zeros(size(timeVector));
    mask = (timeVector >= startTime) & (timeVector <= endTime);

    switch signalType
        case 'Impulse'
            [~, idx] = min(abs(timeVector - startTime));
            signal(idx) = amplitude;

        case 'Step'
            signal(mask) = amplitude;

        case 'Triangular Pulse'
            duration = endTime - startTime;
            mid = startTime + duration/2;
            rising = (timeVector >= startTime) & (timeVector <= mid);
            falling = (timeVector > mid) & (timeVector <= endTime);
            signal(rising) = amplitude * (timeVector(rising) - startTime) / (duration/2);
            signal(falling) = amplitude * (endTime - timeVector(falling)) / (duration/2);

        case 'Rectangular Pulse'
            signal(mask) = amplitude;

        case 'Sawtooth Pulse'
            duration = endTime - startTime;
            signal(mask) = amplitude * mod(timeVector(mask) - startTime, duration/2) / (duration/2);
    end
end

        
        function plotInitialSignals(app)
            % Clear axes
            cla(app.UIAxes1);
            cla(app.UIAxes2);
            cla(app.UIAxes3);
            cla(app.UIAxes4);
            
            % Plot input signal
            if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                stem(app.UIAxes1, app.inputTime, app.inputSignal, 'b', 'filled', 'MarkerSize', 6);
            else
                plot(app.UIAxes1, app.inputTime, app.inputSignal, 'b-');
            end
            title(app.UIAxes1, 'Input Signal');
            xlabel(app.UIAxes1, 'Time');
            ylabel(app.UIAxes1, 'Amplitude');
            grid(app.UIAxes1, 'on');
            
            % Plot impulse response
            if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                stem(app.UIAxes2, app.impulseTime, app.impulseResponse, 'g', 'filled', 'MarkerSize', 6);
            else
                plot(app.UIAxes2, app.impulseTime, app.impulseResponse, 'g-');
            end
            title(app.UIAxes2, 'Impulse Response');
            xlabel(app.UIAxes2, 'Time');
            ylabel(app.UIAxes2, 'Amplitude');
            grid(app.UIAxes2, 'on');
            
            % Initialize convolution result plot
            if app.TabGroup.SelectedTab == app.ContinuousTimeTab
    % Collect all start and end times from input and impulse
    startTimes = [app.InputStartEdit.Value, app.ImpulseStartEdit.Value];
    endTimes = [app.InputEndEdit.Value, app.ImpulseEndEdit.Value];

    % Calculate limits with padding
    minTime = min(startTimes);
    maxTime = max(endTimes);
    padding = (maxTime - minTime) * 0.5;  % 50% padding

    % Handle case where start == end
    if padding == 0
        padding = 1;
    end

    % Final dynamic limits
    leftLimit = minTime - padding;
    rightLimit = maxTime + padding;

    % Apply to all relevant axes
    app.UIAxes1.XLim = [leftLimit, rightLimit];
    app.UIAxes2.XLim = [leftLimit, rightLimit];
    app.UIAxes3.XLim = [leftLimit, rightLimit + (maxTime - minTime)];
    app.UIAxes4.XLim = [leftLimit, rightLimit + (maxTime - minTime)];
end

        end
        
function updateAnimation(app, ~, ~)
    if ~app.isAnimating || app.animationStep >= app.totalSteps
        stop(app.animationTimer);
        app.isAnimating = false;
        return;
    end

    % Clear axes for fresh plotting
    cla(app.UIAxes2);
    cla(app.UIAxes3);

    % Plot shifting impulse response
    if app.TabGroup.SelectedTab == app.DiscreteTimeTab
    % Flip impulse response for convolution
    flippedImpulse = fliplr(app.impulseResponse);

    % Shift amount for current animation step
    shiftAmount = app.animationStep - 1;

    % Compute time indices for the shifted flipped impulse
    shiftedTime = app.inputTime(1) + shiftAmount - (length(flippedImpulse) - 1) : ...
                  app.inputTime(1) + shiftAmount;

    % Plot only if lengths match
    if length(shiftedTime) == length(flippedImpulse)
        stem(app.UIAxes2, shiftedTime, flippedImpulse, 'g', 'filled', 'MarkerSize', 6);
    end
    else
        shiftAmount = (app.animationStep - 1) * app.timeStep;
        mirroredImpulse = flip(app.impulseResponse);
        shiftedTime = app.impulseTime + shiftAmount;
        plot(app.UIAxes2, shiftedTime, mirroredImpulse, 'g-', 'LineWidth', 1.5);
    end
    title(app.UIAxes2, 'Shifting Impulse Response');
    xlabel(app.UIAxes2, 'Time');
    ylabel(app.UIAxes2, 'Amplitude');
    grid(app.UIAxes2, 'on');

    % Remove previous text labels before adding new ones
    delete(findall(app.UIAxes2, 'Type', 'text'));

    % Add dynamic label for End Time at last data point
    x_end = shiftedTime(end);
    y_end = app.impulseResponse(end);
    text(app.UIAxes2, x_end, y_end, ...
        sprintf('End Time: %.2f', x_end), ...
        'VerticalAlignment', 'bottom', ...
        'HorizontalAlignment', 'right', ...
        'FontSize', 10, ...
        'FontWeight', 'bold', ...
        'Color', 'k');

    % Add dynamic label for Amplitude at max amplitude point
    [maxAmp, maxIdx] = max(app.impulseResponse);
    x_amp = shiftedTime(maxIdx);
    y_amp = maxAmp;
    text(app.UIAxes2, x_amp, y_amp, ...
        sprintf('Amplitude: %.2f', maxAmp), ...
        'VerticalAlignment', 'top', ...
        'HorizontalAlignment', 'left', ...
        'FontSize', 10, ...
        'FontWeight', 'bold', ...
        'Color', 'r');

    % Plot convolution result incrementally
    conv_display_len = min(app.animationStep, length(app.convolutionResult));
    if app.TabGroup.SelectedTab == app.DiscreteTimeTab
        stem(app.UIAxes3, app.outputTime(1:conv_display_len), ...
            app.convolutionResult(1:conv_display_len), 'r', 'filled', 'MarkerSize', 6);
    else
        plot(app.UIAxes3, app.outputTime(1:conv_display_len), ...
            app.convolutionResult(1:conv_display_len), 'r-', 'LineWidth', 1.5);
    end
    title(app.UIAxes3, 'Convolution Result');
    xlabel(app.UIAxes3, 'Time');
    ylabel(app.UIAxes3, 'Amplitude');
    grid(app.UIAxes3, 'on');

    % Increment animation step
    app.animationStep = app.animationStep + 1;
end

        function resetAll(app)
            % Stop any running animation
            if ~isempty(app.animationTimer) && isvalid(app.animationTimer)
                stop(app.animationTimer);
            end
            app.isAnimating = false;
            app.animationStep = 1;
            app.showCorrelation = false;
            
            % Clear plots
            cla(app.UIAxes1);
            cla(app.UIAxes2);
            cla(app.UIAxes3);
            cla(app.UIAxes4);
            
            % Reset data
            app.inputSignal = [];
            app.impulseResponse = [];
            app.convolutionResult = [];
            app.correlationResult = [];
            app.inputTime = [];
            app.impulseTime = [];
            app.outputTime = [];
            app.correlationTime = [];
            
            % Hide the 4th axes
            app.UIAxes4.Visible = 'off';
            
            % Reset button text
            app.ComputeCorrButton.Text = 'Show Correlation';
        end
        
        function plotCorrelation(app)
            % Toggle between showing correlation and convolution
            app.showCorrelation = ~app.showCorrelation;
            
            if app.showCorrelation
                % Show correlation plot
                cla(app.UIAxes3);
                
                % Plot correlation result
                if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                    stem(app.UIAxes3, app.correlationTime, app.correlationResult, 'm', 'filled', 'MarkerSize', 6);
                else
                    plot(app.UIAxes3, app.correlationTime, app.correlationResult, 'm-');
                end
                title(app.UIAxes3, 'Cross-Correlation');
                xlabel(app.UIAxes3, 'Lag');
                ylabel(app.UIAxes3, 'Normalized Amplitude');
                grid(app.UIAxes3, 'on');
                
                % Make the 4th axes visible and plot convolution there
                app.UIAxes4.Visible = 'on';
                if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                    stem(app.UIAxes4, app.outputTime, app.convolutionResult, 'r', 'filled', 'MarkerSize', 6);
                else
                    plot(app.UIAxes4, app.outputTime, app.convolutionResult, 'r-');
                end
                title(app.UIAxes4, 'Convolution Result');
                xlabel(app.UIAxes4, 'Time');
                ylabel(app.UIAxes4, 'Amplitude');
                grid(app.UIAxes4, 'on');
                
                % Update correlation button text
                app.ComputeCorrButton.Text = 'Show Convolution';
            else
                % Show convolution plot
                cla(app.UIAxes3);
                
                if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                    stem(app.UIAxes3, app.outputTime, app.convolutionResult, 'r', 'filled', 'MarkerSize', 6);
                else
                    plot(app.UIAxes3, app.outputTime, app.convolutionResult, 'r-');
                end
                title(app.UIAxes3, 'Convolution Result');
                xlabel(app.UIAxes3, 'Time');
                ylabel(app.UIAxes3, 'Amplitude');
                grid(app.UIAxes3, 'on');
                
                % Hide the 4th axes
                app.UIAxes4.Visible = 'off';
                
                % Update correlation button text
                app.ComputeCorrButton.Text = 'Show Correlation';
            end
        end
    end

    % Callbacks that handle component events
    methods (Access = private)

        function startupFcn(app)
            % Initialize animation timer
            app.animationTimer = timer(...
                'ExecutionMode', 'fixedRate', ...
                'Period', 0.05, ...
                'TimerFcn', @app.updateAnimation);
            
            % Set default values
            app.InputStartEdit.Value = 0;
            app.InputEndEdit.Value = 1;
            app.InputAmpEdit.Value = 1;
            app.ImpulseStartEdit.Value = 0;
            app.ImpulseEndEdit.Value = 1;
            app.ImpulseAmpEdit.Value = 1;
            app.SpeedSlider.Value = 50;
            app.AnimationCheckBox.Value = true;
            
            % Set discrete-time defaults
            app.InputVectorEdit.Value = '[1 1 1 1]';
            app.InputIndexEdit.Value = '0';
            app.ImpulseVectorEdit.Value = '[1 1 1]';
            app.ImpulseIndexEdit.Value = '0';
            
            % Position the axes
            app.UIAxes1.Position = [400 550 700 200];
            app.UIAxes2.Position = [400 300 700 200];
            app.UIAxes3.Position = [400 50 350 200];
            app.UIAxes4.Position = [770 50 350 200];
            app.UIAxes4.Visible = 'off';
            
            % Set button text
            app.ComputeCorrButton.Text = 'Show Correlation';
        end

        function ComputeButtonPushed(app, event)
            try
                app.resetAll();
                app.showCorrelation = false;
                
                if app.TabGroup.SelectedTab == app.DiscreteTimeTab
                    app.computeDiscreteConvolution();
                else
                    app.computeContinuousConvolution();
                end
                
                % Prepare for animation if checkbox is checked
                if app.AnimationCheckBox.Value
                    app.animationStep = 1;
                    app.isAnimating = true;
                    
                    % Set animation speed
                    app.animationTimer.Period = max(0.01, (101 - app.SpeedSlider.Value)/1000);
                    start(app.animationTimer);
                else
                    % Just show the final result
                    plotInitialSignals(app);
                end
                
            catch ME
                errordlg(sprintf('Error: %s', ME.message), 'Computation Error');
            end
        end
        
        function ComputeCorrButtonPushed(app, event)
            try
                if isempty(app.inputSignal) || isempty(app.impulseResponse)
                    % If no computation has been done yet, compute first
                    app.ComputeButtonPushed(event);
                end
                
                % Plot the correlation result
                app.plotCorrelation();
                
            catch ME
                errordlg(sprintf('Error: %s', ME.message), 'Computation Error');
            end
        end

        function ResetButtonPushed(app, event)
            app.resetAll();
        end

        function SpeedSliderValueChanged(app, event)
            if app.isAnimating
                app.animationTimer.Period = max(0.01, (101 - app.SpeedSlider.Value)/1000);
            end
        end
        
        function AnimationCheckBoxValueChanged(app, event)
            % No action needed unless you want to do something specific
        end
    end

    % App initialization and construction
    methods (Access = private)

        function createComponents(app)
            % Create UIFigure
            app.UIFigure = uifigure();
            app.UIFigure.Position = [100 100 1200 800];
            app.UIFigure.Name = 'Convolution Visualizer';

            % Create TabGroup
            app.TabGroup = uitabgroup(app.UIFigure);
            app.TabGroup.Position = [20 350 350 400];

            % Create DiscreteTimeTab
            app.DiscreteTimeTab = uitab(app.TabGroup);
            app.DiscreteTimeTab.Title = 'Discrete-Time';

            % Create InputVectorEditLabel
            app.InputVectorEditLabel = uilabel(app.DiscreteTimeTab);
            app.InputVectorEditLabel.Position = [10 300 100 22];
            app.InputVectorEditLabel.Text = 'Input Vector:';

            % Create InputVectorEdit
            app.InputVectorEdit = uieditfield(app.DiscreteTimeTab, 'text');
            app.InputVectorEdit.Position = [120 300 150 22];

            % Create InputIndexEditLabel
            app.InputIndexEditLabel = uilabel(app.DiscreteTimeTab);
            app.InputIndexEditLabel.Position = [10 250 100 22];
            app.InputIndexEditLabel.Text = 'Start Index:';

            % Create InputIndexEdit
            app.InputIndexEdit = uieditfield(app.DiscreteTimeTab, 'text');
            app.InputIndexEdit.Position = [120 250 150 22];

            % Create ImpulseVectorEditLabel
            app.ImpulseVectorEditLabel = uilabel(app.DiscreteTimeTab);
            app.ImpulseVectorEditLabel.Position = [10 200 100 22];
            app.ImpulseVectorEditLabel.Text = 'Impulse Vector:';

            % Create ImpulseVectorEdit
            app.ImpulseVectorEdit = uieditfield(app.DiscreteTimeTab, 'text');
            app.ImpulseVectorEdit.Position = [120 200 150 22];

            % Create ImpulseIndexEditLabel
            app.ImpulseIndexEditLabel = uilabel(app.DiscreteTimeTab);
            app.ImpulseIndexEditLabel.Position = [10 150 100 22];
            app.ImpulseIndexEditLabel.Text = 'Start Index:';

            % Create ImpulseIndexEdit
            app.ImpulseIndexEdit = uieditfield(app.DiscreteTimeTab, 'text');
            app.ImpulseIndexEdit.Position = [120 150 150 22];

            % Create ContinuousTimeTab
            app.ContinuousTimeTab = uitab(app.TabGroup);
            app.ContinuousTimeTab.Title = 'Continuous-Time';

            % Create InputSignalDropDownLabel
            app.InputSignalDropDownLabel = uilabel(app.ContinuousTimeTab);
            app.InputSignalDropDownLabel.Position = [10 270 100 22];
            app.InputSignalDropDownLabel.Text = 'Signal Type:';

            % Create InputSignalDropDown
            app.InputSignalDropDown = uidropdown(app.ContinuousTimeTab);
            app.InputSignalDropDown.Items = {'Impulse', 'Step', 'Triangular Pulse', 'Rectangular Pulse', 'Sawtooth Pulse'};
            app.InputSignalDropDown.Position = [120 270 200 22];
            app.InputSignalDropDown.Value = 'Rectangular Pulse';

            % Create InputStartEditLabel
            app.InputStartEditLabel = uilabel(app.ContinuousTimeTab);
            app.InputStartEditLabel.Position = [10 240 100 22];
            app.InputStartEditLabel.Text = 'Start Time:';

            % Create InputStartEdit
            app.InputStartEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.InputStartEdit.Position = [120 240 200 22];

            % Create InputEndEditLabel
            app.InputEndEditLabel = uilabel(app.ContinuousTimeTab);
            app.InputEndEditLabel.Position = [10 210 100 22];
            app.InputEndEditLabel.Text = 'End Time:';

            % Create InputEndEdit
            app.InputEndEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.InputEndEdit.Position = [120 210 200 22];

            % Create InputAmpEditLabel
            app.InputAmpEditLabel = uilabel(app.ContinuousTimeTab);
            app.InputAmpEditLabel.Position = [10 180 100 22];
            app.InputAmpEditLabel.Text = 'Amplitude:';

            % Create InputAmpEdit
            app.InputAmpEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.InputAmpEdit.Position = [120 180 200 22];

            % Create ImpulseSignalDropDownLabel
            app.ImpulseSignalDropDownLabel = uilabel(app.ContinuousTimeTab);
            app.ImpulseSignalDropDownLabel.Position = [10 140 100 22];
            app.ImpulseSignalDropDownLabel.Text = 'Signal Type:';

            % Create ImpulseSignalDropDown
            app.ImpulseSignalDropDown = uidropdown(app.ContinuousTimeTab);
            app.ImpulseSignalDropDown.Items = {'Impulse', 'Step', 'Triangular Pulse', 'Rectangular Pulse', 'Sawtooth Pulse'};
            app.ImpulseSignalDropDown.Position = [120 140 200 22];
            app.ImpulseSignalDropDown.Value = 'Triangular Pulse';

            % Create ImpulseStartEditLabel
            app.ImpulseStartEditLabel = uilabel(app.ContinuousTimeTab);
            app.ImpulseStartEditLabel.Position = [10 110 100 22];
            app.ImpulseStartEditLabel.Text = 'Start Time:';

            % Create ImpulseStartEdit
            app.ImpulseStartEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.ImpulseStartEdit.Position = [120 110 200 22];

            % Create ImpulseEndEditLabel
            app.ImpulseEndEditLabel = uilabel(app.ContinuousTimeTab);
            app.ImpulseEndEditLabel.Position = [10 80 100 22];
            app.ImpulseEndEditLabel.Text = 'End Time:';

            % Create ImpulseEndEdit
            app.ImpulseEndEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.ImpulseEndEdit.Position = [120 80 200 22];

            % Create ImpulseAmpEditLabel
            app.ImpulseAmpEditLabel = uilabel(app.ContinuousTimeTab);
            app.ImpulseAmpEditLabel.Position = [10 50 100 22];
            app.ImpulseAmpEditLabel.Text = 'Amplitude:';

            % Create ImpulseAmpEdit
            app.ImpulseAmpEdit = uieditfield(app.ContinuousTimeTab, 'numeric');
            app.ImpulseAmpEdit.Position = [120 50 200 22];

            % Create ComputeButton
            app.ComputeButton = uibutton(app.UIFigure, 'push');
            app.ComputeButton.ButtonPushedFcn = createCallbackFcn(app, @ComputeButtonPushed, true);
            app.ComputeButton.Position = [20 310 160 30];
            app.ComputeButton.Text = 'Compute Convolution';

            % Create ComputeCorrButton
            app.ComputeCorrButton = uibutton(app.UIFigure, 'push');
            app.ComputeCorrButton.ButtonPushedFcn = createCallbackFcn(app, @ComputeCorrButtonPushed, true);
            app.ComputeCorrButton.Position = [20 270 140 30];
            app.ComputeCorrButton.Text = 'Show Correlation';

            % Create ResetButton
            app.ResetButton = uibutton(app.UIFigure, 'push');
            app.ResetButton.ButtonPushedFcn = createCallbackFcn(app, @ResetButtonPushed, true);
            app.ResetButton.Position = [20 230 140 30];
            app.ResetButton.Text = 'Reset';

            % Create SpeedSliderLabel
            app.SpeedSliderLabel = uilabel(app.UIFigure);
            app.SpeedSliderLabel.Position = [20 180 120 22];
            app.SpeedSliderLabel.Text = 'Animation Speed:';

            % Create SpeedSlider
            app.SpeedSlider = uislider(app.UIFigure);
            app.SpeedSlider.Limits = [1 100];
            app.SpeedSlider.ValueChangedFcn = createCallbackFcn(app, @SpeedSliderValueChanged, true);
            app.SpeedSlider.Position = [20 160 310 3];
            app.SpeedSlider.Value = 50;

            % Create AnimationCheckBox
            app.AnimationCheckBox = uicheckbox(app.UIFigure);
            app.AnimationCheckBox.Text = 'Show Animation';
            app.AnimationCheckBox.Position = [20 100 150 22];
            app.AnimationCheckBox.Value = true;
            app.AnimationCheckBox.ValueChangedFcn = createCallbackFcn(app, @AnimationCheckBoxValueChanged, true);

            % Create UIAxes1
            app.UIAxes1 = uiaxes(app.UIFigure);
            title(app.UIAxes1, 'Input Signal')
            xlabel(app.UIAxes1, 'Time')
            ylabel(app.UIAxes1, 'Amplitude')
            app.UIAxes1.Position = [350 400 800 200];

            % Create UIAxes2
            app.UIAxes2 = uiaxes(app.UIFigure);
            title(app.UIAxes2, 'Impulse Response')
            xlabel(app.UIAxes2, 'Time')
            ylabel(app.UIAxes2, 'Amplitude')
            app.UIAxes2.Position = [350 250 800 200];

            % Create UIAxes3
            app.UIAxes3 = uiaxes(app.UIFigure);
            title(app.UIAxes3, 'Convolution Result')
            xlabel(app.UIAxes3, 'Time')
            ylabel(app.UIAxes3, 'Amplitude')
            app.UIAxes3.Position = [350 50 400 200];

            % Create UIAxes4
            app.UIAxes4 = uiaxes(app.UIFigure);
            title(app.UIAxes4, 'Correlation Result')
            xlabel(app.UIAxes4, 'Lag')
            ylabel(app.UIAxes4, 'Amplitude')
            app.UIAxes4.Position = [750 50 400 200];
            app.UIAxes4.Visible = 'off';
        end
    end

    methods (Access = public)
        function app = ConvolutionVisualizer()
            createComponents(app)
            registerApp(app, app.UIFigure)
            runStartupFcn(app, @startupFcn)
            if nargout == 0
                clear app
            end
        end

        function delete(app)
            delete(app.UIFigure)
            if ~isempty(app.animationTimer) && isvalid(app.animationTimer)
                stop(app.animationTimer);
                delete(app.animationTimer);
            end
        end
    end
end

