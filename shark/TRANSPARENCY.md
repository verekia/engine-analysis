# Transparency

This topic was not explicitly specified in the original shark implementation proposal.

Standard approaches for transparency rendering include:
- Depth sorting of transparent objects (back-to-front)
- Separate render pass for transparent geometry
- Alpha blending with appropriate blend functions
- Two-pass rendering for transparent objects (depth write pass + color pass)
