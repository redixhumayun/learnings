# Seon Ho Kim - Spatial-Visual Query Research

## Overview
Research at USC (IMSC) focuses on **spatial-visual queries** - combining spatial and visual features for searching geo-tagged images/videos.

## Core Challenge
Geo-tagged media has two data types:
- **Spatial**: Low-dimensional (lat/lon)
- **Visual**: High-dimensional (CNN features, e.g., 4096 dims)

Traditional indexes struggle with the visual dimension due to "curse of dimensionality."

## Approach: R*-tree Based Indexes

### Key Insight
**Spatial locality of visual features**: Nearby street images tend to look similar. This allows spatial pruning to implicitly filter visual relevance.

### Index Variants
1. **SRI** (Spatial R*-tree Index): Organize by location, augment leaves with visual features
2. **VRI** (Visual R*-tree Index): Organize by reduced-dim visual features, augment with spatial
3. **PSV** (Plain Spatial-Visual R*-tree): Concatenate spatial + visual descriptors into single MBR
4. **ASV** (Adaptive Spatial-Visual R*-tree): Separate MBRs for spatial and visual, with adapted optimization criteria

### Hybrid Approaches
Two-level structures combining R*-tree + LSH:
- **Spatial-First Index**: R*-tree primary, LSH secondary
- **Visual-First Index**: LSH primary, R*-tree secondary

## Two Types of Inaccuracy
1. **Spatial inaccuracy**: GPS position ≠ actual scene location
2. **Visual inaccuracy**: Dimensionality reduction causes false negatives

## Key Papers
- ICDE 2020: "A Class of R*-tree Indexes for Spatial-Visual Search of Geo-tagged Street Images"
- ACM MM 2017: "Hybrid Indexes for Spatial-Visual Search"

## Projects
- **MediaQ**: Social media data collection/management
- **TVDP**: Translational Visual Data Platform for urban geo-tagged data

## Key Collaborators
- Cyrus Shahabi (IMSC Director)
- Abdullah Alfarrarjeh (PhD, now at Google)
