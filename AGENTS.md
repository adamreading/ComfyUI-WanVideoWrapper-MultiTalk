# AGENTS.md - MultiTalk Progressive Reference Frame Implementation

## Project Overview

**Objective**: Implement progressive reference frame conditioning in ComfyUI-WanVideoWrapper's MultiTalk branch to eliminate visual jumps that occur every 81 frames during long video generation.

**Problem**: MultiTalk currently reverts to the original reference image at each context window boundary, causing jarring visual discontinuities. The solution requires extracting latent representations from completed context windows and using them as progressive reference conditioning for subsequent windows.

**Target**: Transform 253-frame MultiTalk generation from discrete 81-frame segments with visual jumps into continuous temporal evolution.

## Technical Context

### System Environment
- **Hardware**: AMD 9950X, RTX 5090, 96GB RAM, WSL2 (Windows 11 Pro)
- **Framework**: ComfyUI with WanVideoWrapper
- **Model**: MultiTalk (audio-driven video generation)
- **Context**: 81-frame windows with 32-frame overlap using static_standard scheduler

### Repository Structure
```
ComfyUI-WanVideoWrapper/
├── nodes.py (root) - Main ComfyUI nodes including WanVideoSampler
├── context.py - Context window scheduling (already functional)
├── multitalk/
│   ├── multitalk.py - Core attention mechanisms and conditioning
│   ├── nodes.py - MultiTalk-specific nodes
│   └── wav2vec2.py - Audio processing
```

### Current Working System
- **Context Options**: 81 frames, 32 overlap, stride 4, static_standard scheduler
- **Sliding Windows**: Properly implemented with overlapping frame processing
- **Issue**: Reference conditioning resets to original image every 81 frames instead of using progressive frames

## Implementation Strategy

### Phase 1: Latent Extraction in Sampler (nodes.py root)

**Target Location**: `WanVideoSampler` class, specifically the context window processing section:
```python
if context_options is not None:
    def create_window_mask(noise_pred_context, c, latent_video_length, context_overlap, looped=False):
```

**Required Modifications**:
1. **Context Boundary Detection**: Add logic to detect completion of each 81-frame context window
2. **Latent Buffer System**: Implement buffer to store latent representations from frames 75-80 of completed windows
3. **Progressive Reference Injection**: Modify conditioning pipeline to use extracted latents instead of original reference image

**Key Technical Requirements**:
- Work directly in latent space (no decode/encode cycles)
- Maintain compatibility with existing context overlap system
- Preserve motion vectors and temporal coherence
- Efficient VRAM management for RTX 5090

### Phase 2: Context Integration (context.py)

**Current System**: `static_standard` scheduler already provides proper sliding windows
**Enhancement Needed**: Coordinate with progressive reference extraction timing
**Parameters to Preserve**: context_size=81, context_overlap=32, delta=49

### Phase 3: MultiTalk Conditioning (multitalk/multitalk.py)

**Target Classes**: 
- `SingleStreamAttention` and `SingleStreamMultiAttention`
- `calculate_x_ref_attn_map` function

**Modifications Required**:
- Accept progressive latent references instead of static image conditioning
- Maintain compatibility with audio conditioning pipeline
- Preserve existing attention mechanisms while adding dynamic reference capability

### Phase 4: Node Interface (multitalk/nodes.py)

**New Node Requirements**:
- Progressive reference toggle (enable/disable)
- Reference frame selection range configuration (default: frames 75-80)
- Latent buffer size management
- Backward compatibility with existing workflows

## Development Guidelines

### Code Quality Standards
- **No Breaking Changes**: Maintain full backward compatibility with existing MultiTalk workflows
- **Memory Efficiency**: Optimize for RTX 5090 VRAM constraints
- **Error Handling**: Robust error handling for context boundary edge cases
- **Documentation**: Inline comments explaining progressive reference logic

### Testing Requirements
- **Unit Tests**: Test each component independently
- **Integration Tests**: Validate complete pipeline with 162-frame sequences before full 253-frame tests
- **Performance Benchmarks**: Monitor VRAM usage and generation speed impact
- **Quality Metrics**: Implement continuity assessment across context boundaries

### Implementation Priorities
1. **Latent extraction accuracy** - Ensure clean latent capture from context windows
2. **Memory management** - Prevent VRAM overflow with efficient buffer system
3. **Temporal coherence** - Maintain motion vectors and visual consistency
4. **User experience** - Simple toggle for progressive reference mode

## Technical Specifications

### Latent Space Operations
- **Format**: Work with VAE-encoded latent representations
- **Selection Strategy**: Extract latents from frames 75-80 (optimal stability/progression balance)
- **Storage**: Temporary buffer system with automatic cleanup
- **Injection**: Replace original reference conditioning at context boundaries

### Context Window Coordination
- **Timing**: Extract latents immediately after context window completion
- **Overlap Handling**: Coordinate with existing 32-frame overlap system
- **Boundary Management**: Smooth transitions between extracted reference latents

### Memory Management
- **Buffer Size**: Store only essential latent information for next context window
- **Cleanup Strategy**: Automatic disposal of unused latent data
- **VRAM Monitoring**: Track memory usage to prevent overflow

## Expected Outcomes

### Success Criteria
- **Visual Continuity**: Elimination of 81-frame visual jumps in MultiTalk generation
- **Temporal Coherence**: Smooth character progression through 253-frame sequences
- **Performance Maintenance**: No significant impact on generation speed or VRAM usage
- **Compatibility**: Existing workflows continue to function without modification

### Validation Methods
- **Before/After Comparison**: Direct comparison of 253-frame outputs
- **Continuity Metrics**: Quantitative assessment of visual coherence across context boundaries
- **User Testing**: Validation with various MultiTalk scenarios and configurations

## Development Notes

### Known Constraints
- Cannot modify underlying MultiTalk model architecture
- Must work within existing ComfyUI node framework
- Preserve audio conditioning pipeline functionality
- Maintain compatibility with TeaCache and other extensions

### Architecture Decisions
- **Sampler-Level Implementation**: All progressive reference logic contained within WanVideoSampler
- **Latent-Based Approach**: Use latent representations instead of decoded frames
- **Optional Enhancement**: Progressive reference as configurable feature, not mandatory change

### Risk Mitigation
- **Incremental Development**: Test each phase independently before integration
- **Fallback Options**: Maintain ability to disable progressive reference if issues arise
- **Performance Monitoring**: Continuous assessment of memory and speed impact

## File Modification Checklist

### Primary Files
- [ ] `nodes.py` (root) - WanVideoSampler modifications
- [ ] `multitalk/multitalk.py` - Attention mechanism updates
- [ ] `multitalk/nodes.py` - New progressive reference node

### Secondary Files
- [ ] `context.py` - Coordination enhancements (if needed)
- [ ] Documentation updates
- [ ] Example workflow files

### Testing Files
- [ ] Unit test implementations
- [ ] Integration test scenarios
- [ ] Performance benchmark scripts

This AGENTS.md provides comprehensive guidance for implementing the progressive reference frame solution while maintaining the existing ComfyUI-WanVideoWrapper functionality and optimizing for the specified hardware environment.
